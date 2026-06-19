---
id: 019eddaf-ffb3-71e9-a322-c4b728be4d54
title: HTTP 타임아웃과 분산 시스템의 부분 실패
topics:
  - http
  - timeout
  - 분산 시스템
  - two-generals-problem
  - 멱등성
  - idempotency
  - 재시도
  - 네트워크
sources:
  - 019eddae-b53b-77fd-8c52-e20d36f14bce
created_at: '2026-06-19T02:22:39.539Z'
updated_at: '2026-06-19T02:22:39.539Z'
---
## TL;DR

**A timeout is just "the moment the client gave up waiting" — it does NOT mean "the server didn't do the work."** The request already reached the server and was processed normally, but the client decides it's a failure *because the response didn't arrive in time*.

---

## 1. Where exactly does the timeout fire?

Breaking down an HTTP request's lifecycle at the TCP level:

```
[Client]                                        [Server]
   |  (1) SYN  ───────────────────────────────►  |   ← connection timeout
   |  (2) ◄─────────────────── SYN-ACK ────────  |
   |  (3) ACK + HTTP Request ─────────────────►  |   ← request body fully sent
   |                                             |
   |                            [server processes] |   ← side effects like DB write, payment, point credit already executed
   |                                             |
   |  (4) ◄────────────── HTTP Response ───────  |   ← read/socket timeout (fires if not received here)
```

Most "network timeouts" are the **read timeout (= socket/response timeout) at step (4)**. In other words:

- At (3), **the request has already fully reached the server**, and
- The server received it and **already executed side effects like DB writes, payments, point credits**, but
- The response in (4) took longer than the timeout threshold to come back to the client

At that moment, from the client's perspective it's a "failure," but from the server's perspective it's a "success." **A state where the two nodes believe different facts** results.

---

## 2. Why does only the response get delayed or lost — concrete scenarios

### Scenario A: Server processing time > client timeout
The most common. Client timeout is 3s but server processing takes 3.5s.
- The server commits the transaction normally over 3.5s
- The client closes the socket at 3s and handles the error
- The server tries to write the response at step (4), but the socket is already closed → only `EPIPE`/`Broken pipe` is logged, and **the already-committed DB is NOT rolled back**

### Scenario B: Only the response packet is lost
The request went fine but there's packet loss/routing issue on the return path. TCP attempts retransmission, but the client timeout fires first.

### Scenario C: Intermediate proxy/LB timeout mismatch
```
client(30s) → ALB/Nginx(60s) → app server(processing 90s)
```
- Client or LB cuts off first (504 Gateway Timeout etc.)
- But the backend app server **finishes processing to the end, unaware that the connection was severed**
- Always occurs when Nginx's `proxy_read_timeout` or ALB's idle timeout is shorter than backend processing time

### Scenario D: Queue/async processing
A structure that enqueues to a message queue on receipt and waits for a synchronous response, where the response (ack) comes back late. Same thing happens with the RabbitMQ RPC pattern (`hasResponse: true`) — the consumer finished processing but the reply to the reply queue is delayed.

### Scenario E: Connection pool exhaustion / GC pauses
*Right after* processing, the server hits a Stop-the-world GC or event loop block, delaying the response flush.

---

## 3. Root cause — the "Two Generals Problem"

This isn't a network bug — it's a **theoretically unavoidable problem**.

> In an asynchronous network, the sender cannot ever distinguish whether "no response came" is because **the request never arrived** or because **only the response didn't come back.**

The only information the client receives is "no response by timeout," and from this single observation, the following 4 cases all look identical:

| Actual server state | What the client sees |
|---|---|
| Request never even reached the server | Timeout |
| Reached, but died before processing | Timeout |
| **Processing complete, only the response was lost** ← your case | Timeout |
| Processing complete, connection broken mid-response-send | Timeout |

The client has **no way in principle to distinguish** these 4 cases. That's why "timeout fired but the server succeeded" happens.

---

## 4. Why this is dangerous (operational impact)

If the client sees the timeout and says "it failed, let's retry":

- **Duplicate execution**: points credited twice, payment twice, wallet created twice
- **State inconsistency**: client DB says "failed," server DB says "succeeded"

The branch currently being worked on (`potluck point request when user wallet doesn't exist`) also seems to be in the context of dealing with this kind of partial failure / state inconsistency.

---

## 5. Solutions — treat the timeout as "might be a success"

### (1) [[Idempotency]] is the answer
Make it so that retrying produces the same result as doing it once:
- Client sends an `Idempotency-Key` (UUID) as a header
- Server checks whether that key has already been processed → if so, **returns the stored response as-is** (does not re-execute)
- Almost mandatory for side-effecting APIs like payments/points

```typescript
// conceptual example
async execute(idempotencyKey: string, payload: PointRequest) {
  const existing = await this.repo.findByKey(idempotencyKey);
  if (existing) return existing.response; // do NOT re-execute, return stored result
  // ... actually process and store result
}
```

### (2) Retry only when idempotent
Blindly retrying a non-idempotent POST is unsafe. Without an idempotency key, "no retry on timeout + check status via a separate query" is the safe path.

### (3) Align timeout layers
Not `client > LB > nginx > app`, but **the outer layers must be the longest**:
```
app processing SLA (e.g. 5s) < nginx proxy_read_timeout (10s) < LB idle (15s) < client (20s)
```
This way an intermediate node never severs a backend that's still processing.

### (4) Reconciliation API for status check
Provide a query endpoint so the client can ask after a timeout: "what happened to that request I just sent?"

### (5) Switch to async
For long operations, give up on a synchronous response — return `202 Accepted + job ID` immediately, deliver the result via polling/webhook.

---

## Key takeaway

A timeout is **not a network signal, it's the limit of the client's patience.** The request already reached the server at step (3) and caused side effects — the response at step (4) was merely late or lost. Because of the [[Two Generals Problem]], this is *unavoidable in principle*, so the answer is not "block it" but **"make it safe with idempotency."**

---

## 한국어

### 한 줄 요약

**타임아웃은 "클라이언트가 기다리기를 포기한 시점"일 뿐, "서버가 일을 안 했다"는 뜻이 아니다.** 요청은 이미 서버에 도착해서 정상 처리됐는데, 클라이언트는 *응답을 제때 못 받았기 때문에* 실패로 판단하는 것입니다.

---

### 1. 타임아웃이 정확히 "어디서" 터지는가

HTTP 요청의 생애주기를 TCP 단위로 쪼개보면:

```
[클라이언트]                                    [서버]
   |  (1) SYN  ───────────────────────────────►  |   ← connection timeout
   |  (2) ◄─────────────────── SYN-ACK ────────  |
   |  (3) ACK + HTTP Request ─────────────────►  |   ← 요청 본문 전송 완료
   |                                             |
   |                            [서버가 요청 처리] |   ← DB write, 결제, 포인트 지급 등 실제 부수효과 발생
   |                                             |
   |  (4) ◄────────────── HTTP Response ───────  |   ← read/socket timeout (여기서 못 받으면 터짐)
```

대부분의 "네트워크 타임아웃"은 **(4) 단계의 read timeout(=socket/response timeout)** 입니다. 즉:

- (3)에서 **요청은 이미 서버에 완전히 도달**했고
- 서버는 그걸 받아서 **DB 쓰기, 결제, 포인트 지급 같은 부수효과(side effect)를 이미 실행**했는데
- (4)의 응답이 클라이언트에게 돌아오는 시간이 타임아웃 임계값을 넘긴 것

이 순간 클라이언트 입장에서는 "실패"지만 서버 입장에서는 "성공"입니다. **두 노드가 서로 다른 사실을 믿게 되는 상태**가 됩니다.

---

### 2. 왜 응답만 늦거나 사라지는가 — 구체적 시나리오

#### 시나리오 A: 서버 처리 시간 > 클라이언트 타임아웃
가장 흔합니다. 클라이언트 타임아웃이 3초인데 서버 처리가 3.5초 걸림.
- 서버는 3.5초 동안 정상적으로 트랜잭션을 커밋
- 클라이언트는 3초에 소켓을 끊고 에러 처리
- 서버는 4단계에서 응답을 쓰려 했지만 소켓이 이미 닫혀 있음 → `EPIPE`/`Broken pipe` 만 로그에 남고, **이미 커밋된 DB는 롤백 안 됨**

#### 시나리오 B: 응답 패킷만 유실
요청은 잘 갔는데 돌아오는 경로에서 패킷 손실/라우팅 문제. TCP가 재전송을 시도하지만 클라이언트 타임아웃이 먼저 만료.

#### 시나리오 C: 중간 프록시/LB의 타임아웃 불일치
```
client(30s) → ALB/Nginx(60s) → app server(처리 90s)
```
- 클라이언트나 LB가 먼저 끊어버림 (504 Gateway Timeout 등)
- 그러나 백엔드 앱 서버는 연결 끊김을 **모른 채 끝까지 처리**를 완료
- Nginx의 `proxy_read_timeout`, ALB의 idle timeout이 백엔드 처리시간보다 짧으면 항상 발생

#### 시나리오 D: 큐/비동기 처리
요청을 받으면 메시지 큐에 enqueue하고 동기 응답을 기다리는 구조에서, 응답(ack)이 늦게 오는 경우. RabbitMQ RPC 패턴(`hasResponse: true`)에서도 consumer가 처리는 끝냈는데 reply 큐로 가는 응답이 지연되면 동일.

#### 시나리오 E: 커넥션 풀 고갈 / GC 멈춤
서버가 처리 *직후* Stop-the-world GC나 이벤트 루프 블로킹으로 응답 flush가 지연.

---

### 3. 근본 원인 — "Two Generals Problem"

이건 네트워크 버그가 아니라 **이론적으로 회피 불가능한 문제**입니다.

> 비동기 네트워크에서, 메시지를 보낸 쪽은 "응답이 안 온 것"이 **요청이 안 갔기 때문인지**, **응답이 안 온 것뿐인지** 절대 구분할 수 없다.

클라이언트가 받는 정보는 "타임아웃까지 응답 없음" 단 하나뿐이고, 이 한 가지 관측으로부터 다음 4가지가 전부 똑같이 보입니다:

| 실제 서버 상태 | 클라이언트가 보는 것 |
|---|---|
| 요청이 서버에 도달조차 안 함 | 타임아웃 |
| 도달했지만 처리 전에 죽음 | 타임아웃 |
| **처리 완료, 응답만 유실** ← 당신의 케이스 | 타임아웃 |
| 처리 완료, 응답 전송 중 끊김 | 타임아웃 |

클라이언트는 이 4가지를 **구별할 방법이 원리적으로 없습니다.** 그래서 "타임아웃 났는데 서버는 성공"이 발생하는 겁니다.

---

### 4. 이게 왜 위험한가 (실무 영향)

클라이언트가 타임아웃을 보고 "실패했네, 재시도하자"라고 하면:

- **중복 실행**: 포인트 2번 지급, 결제 2번, 지갑 2번 생성
- **상태 불일치**: 클라이언트 DB는 "실패", 서버 DB는 "성공"

지금 작업 중인 브랜치(`potluck point request when user wallet doesn't exist`)도 결국 이런 부분 실패/상태 불일치를 다루는 맥락으로 보입니다.

---

### 5. 해결책 — 타임아웃을 "성공일 수도 있는 상태"로 다루기

#### (1) [[멱등성]](Idempotency)이 정답
재시도해도 결과가 한 번 한 것과 같게 만드는 것:
- 클라이언트가 `Idempotency-Key`(UUID)를 헤더로 보냄
- 서버는 그 키로 이미 처리된 요청인지 확인 → 처리됐으면 **저장된 응답을 그대로 반환** (재실행 안 함)
- 결제/포인트 같은 부수효과 API는 거의 필수

```typescript
// 개념 예시
async execute(idempotencyKey: string, payload: PointRequest) {
  const existing = await this.repo.findByKey(idempotencyKey);
  if (existing) return existing.response; // 재실행 금지, 저장된 결과 반환
  // ... 실제 처리 후 결과 저장
}
```

#### (2) 재시도는 멱등할 때만
멱등하지 않은 POST를 무지성 재시도하면 안 됩니다. 멱등 키 없이는 "타임아웃 시 재시도 금지 + 별도 조회로 상태 확인"이 안전.

#### (3) 타임아웃 계층 정렬
`client > LB > nginx > app`이 아니라 **바깥쪽이 가장 길게**:
```
app 처리 SLA(예 5s) < nginx proxy_read_timeout(10s) < LB idle(15s) < client(20s)
```
이렇게 해야 중간 노드가 "처리 중인" 백엔드를 먼저 끊어버리는 일이 없습니다.

#### (4) 상태 확인(Reconciliation) API
타임아웃 후 클라이언트가 "방금 그 요청 어떻게 됐어?"를 물어볼 수 있는 조회 엔드포인트 제공.

#### (5) 비동기로 전환
오래 걸리는 작업은 동기 응답을 포기하고 `202 Accepted + 작업 ID`를 즉시 반환, 결과는 polling/webhook으로.

---

### 핵심 정리

타임아웃은 **네트워크 신호가 아니라 클라이언트의 인내심 한계**입니다. 요청은 (3)단계에서 이미 서버에 도달해 부수효과를 일으켰고, 단지 (4)단계 응답이 늦거나 유실됐을 뿐이죠. 이건 [[Two Generals Problem]] 때문에 *원리적으로 회피 불가능*하므로, "막는다"가 아니라 **멱등성으로 안전하게 만든다**가 정답입니다.
