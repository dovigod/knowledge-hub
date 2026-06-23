---
id: 019ef3b4-aa99-71a1-a586-0c1e62796c15
title: '네트워크 정책 레이어: L4 vs L7'
topics:
  - network-policy
  - l4
  - l7
  - osi
tags:
  - networking
  - kubernetes
  - cilium
  - ztunnel
  - mtls
  - ebpf
  - security
  - education
sources:
  - 019ed89a-6333-736b-a257-45a8581f1a66
created_at: '2026-06-23T08:59:24.185Z'
updated_at: '2026-06-23T08:59:24.185Z'
---
> [!tip] TL;DR
> 네트워크 정책은 **L3/L4(IP·포트)가 기본**이고 모든 [[CNI]]가 지원합니다. 문제는 L4가 "포트 열림/닫힘" 수준이라 같은 8080 포트로 들어오는 정상 `GET`과 위험한 `DELETE`를 못 가른다는 것. **L7 정책**은 패킷 내용을 파싱해 "용건 단위"로 자릅니다. 그러나 [[Cilium]]/[[Envoy]] 같은 L7 프록시가 **그 프로토콜을 평문으로 읽고 이해할 수 있을 때만** 가능합니다. [[PostgreSQL]]/[[MySQL]]/[[Redis]] 같은 DB는 전용 바이너리 와이어 프로토콜이라 파서가 없고, [[mTLS]]로 암호화된 트래픽은 내용이 암호문이라 못 읽습니다. 그래서 이런 트래픽은 **L4가 "더 적합"한 게 아니라 "유일하게 가능한 선택"**입니다. 실무는 L4로 굵게 거르고 L7로 정밀하게 거르는 식으로 겹쳐 씁니다.

## 먼저: 네트워크 레이어와 "각 레이어에서 볼 수 있는 것"

```
L7 (애플리케이션)  HTTP 경로·메서드·헤더    "GET /api/users"
L6 (표현)         (TLS도 여기 걸침)        
L5 (세션)
L4 (전송)          포트·프로토콜            "TCP 8080"
L3 (네트워크)      IP 주소                  "10.0.0.5 → 10.0.0.8"
L2 (데이터링크)    MAC 주소
L1 (물리)          전기 신호·케이블
```

위로 갈수록 **"내용에 가까운 정보"**, 아래로 갈수록 **"배달 경로에 가까운 정보"**. 네트워크 정책은 보통 **L3/L4 기준이 기본**이고 **L7은 그 위에 얹는 것**이라는 점을 먼저 못 박아 두면 헷갈리지 않습니다.

## 같은 통신을 레이어별 정책으로 표현하면

상황: "프론트엔드 → 백엔드 API 통신을 제어하고 싶다"

### L3 정책 (IP 기준)

```yaml
# 의사 코드
from:
  ipBlock: 10.0.1.0/24
action: allow
```
→ "이 동네 사람만 출입 가능"

### L4 정책 (포트/프로토콜 기준) — 가장 흔함

```yaml
ports:
  - port: 8080
    protocol: TCP
action: allow
```
→ "정문(8080)으로만 들어와, 다른 문은 잠김"

### L7 정책 (HTTP 내용 기준)

```yaml
http:
  - method: GET
    path: /api/products
    action: allow
  - method: DELETE
    path: /api/users/*
    action: deny
  - headers:
      Authorization: "Bearer .*"
    action: allow
```
→ "들어오는 건 되는데, **무슨 용건이냐**까지 보고 통과/차단"

## L7이 왜 좋은가 — L4의 결정적 한계

L4 정책의 본질적 한계: **포트는 열거나 닫거나 둘 중 하나**입니다.

```
L4의 세계: TCP 8080 = 열림 ✅  또는  닫힘 ❌
           → 한번 열면 그 포트로 오는 "모든 요청"이 통과
```

문제는, **같은 포트(8080)로 안전한 요청과 위험한 요청이 섞여 들어온다**는 것:

```
TCP 8080 (전부 같은 포트)
   ├─ GET    /api/products        ← 정상 조회 (허용하고 싶음)
   ├─ GET    /api/users           ← 개인정보 조회 (인증 필요)
   ├─ POST   /api/orders          ← 주문 생성 (인증 필요)
   └─ DELETE /api/users/{id}      ← 계정 삭제 (관리자만)
```

> [!warning] L4의 전부 아니면 전무
> **L4로는 이걸 구분 못 합니다.** 8080을 열면 위 네 개가 다 통과, 닫으면 다 차단. 같은 포트 안에서 "조회는 허용, 삭제는 차단"처럼 자르려면 **반드시 L7**.

### 비유 — 경비원의 시야

| | L4 경비 | L7 경비 |
|---|---|---|
| 위치 | 건물 정문 | 각 사무실 앞 |
| 확인 | 출입증(포트) 유무 | 출입증 + **무슨 용무로 왔는지** |
| 한계 | 들어오면 안에서 뭐든 가능 | 행동 하나하나(요청 단위) 통제 |

[[Cilium]]은 [[eBPF]]로 패킷 내용을 커널에서 파싱할 수 있어 L7 정책이 가능합니다. iptables 기반 [[Calico]]는 HTTP 내용까지 들여다보기 어렵기 때문에 L3/L4까지가 한계입니다.

## 그럼 L4는 필요 없나? — 둘 다 겹쳐 씁니다 (defense in depth)

> [!note] L4 정책은 사라지지 않습니다
> L7만 쓰는 게 아니라 **계층적으로 겹쳐서** 씁니다. L7이 좋다고 L4를 빼는 게 아니라, L4 위에 L7을 추가로 얹습니다.

```
1차 (L3/L4): "그 동네(IP)에서, 정문(8080)으로만"   ← 굵은 필터, 빠름
        ▼ 통과한 패킷만
2차 (L7):    "그중에서도 GET만, DELETE는 차단"      ← 정밀 필터, 비쌈
```

이유:

| 이유 | 설명 |
|---|---|
| **성능** | L7 검사는 패킷 페이로드 파싱이라 비쌈. L4로 먼저 99% 거르고 통과한 것만 L7로 검사하는 게 효율적. |
| **적용 범위** | 모든 트래픽이 HTTP는 아님 — DB·암호화·일반 TCP 스트림은 L7 자체가 불가능 (아래에서 상세). |
| **방어 심층화** | 한 겹 뚫려도 다음 겹이 막음. L7 정책 설정 실수가 있어도 L4가 1차 방어선. |

## "L4가 더 적합" — 진짜 의미

> [!warning] "더 적합"의 정정
> 원래 "DB·gRPC 등은 L4가 더 적합"이라고 말했지만 정확하지 않습니다. **[[gRPC]]는 사실 L7 정책이 됩니다** ([[HTTP/2]] 위에서 도니까 `POST /package.Service/Method` 단위로 제어 가능). 진짜 L4가 적합한 건 **"프록시가 내용을 못 읽는 트래픽"**입니다.

L7 정책이 성립하려면 두 가지가 필요합니다:

1. **프록시가 그 프로토콜을 해석할 줄 알아야** 함 ([[Cilium]]은 HTTP, gRPC, Kafka, DNS 파서 보유)
2. 트래픽이 **평문**이거나, 프록시가 **복호화할 수 있어야** 함

이 둘 중 하나라도 안 되면 L7은 원천 불가능하고, **L4로만** 제어할 수 있습니다.

## L4밖에 안 되는(=L4가 "적합한") 4가지 경우

### (1) 프록시가 모르는 바이너리 프로토콜 — DB가 대표

[[PostgreSQL]], [[MySQL]], [[Redis]] 같은 DB는 각자 **고유 바이너리 와이어 프로토콜**을 씁니다. HTTP처럼 표준화된 "경로/메서드" 개념이 없어요.

```
HTTP:
  GET /api/users HTTP/1.1\r\n
  Host: api.example.com\r\n
  ← 텍스트, 구조 명확, 파싱 쉬움

PostgreSQL 와이어 프로토콜:
  0x51 0x00 0x00 0x00 0x1a 0x53 0x45 0x4c ...
  ← 메시지 타입 바이트 + 길이 + 페이로드 (전용 바이너리)

Redis RESP:
  *3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
  ← 자기만의 텍스트 프로토콜이지만 HTTP가 아님
```

[[Cilium]]의 L7 파서는 HTTP/gRPC/Kafka/DNS 정도만 이해합니다. **PostgreSQL 쿼리를 파싱해서 "`SELECT`는 허용, `DROP TABLE`은 차단" 같은 걸 못 합니다** — 그 프로토콜 파서가 없으니까. 그래서 DB 트래픽은 사실상 **L4로 "이 Pod만 5432 포트 접속 허용"** 정도가 최선:

```
# DB 트래픽 정책의 현실
from:
  podSelector:
    app: api-server         ← "누가" (L3/identity)
to:
  ports:
    - port: 5432            ← "어디" (L4)
      protocol: TCP
# "무엇을 쿼리하는지"는 알 수 없음 → 정책 불가
```

> [!example] DB 전용 프록시는 별도 존재
> "SQL 단위 정책"을 정말 원한다면 [[ProxySQL]], [[Vitess]] 같은 **DB 전용 프록시/게이트웨이**를 별도로 둬야 합니다. 일반 [[service mesh]]·[[CNI]] 레이어에서는 안 됩니다.

### (2) 암호화된 트래픽 — 내용이 안 보임

[[TLS]]/[[mTLS]]로 암호화된 트래픽은 프록시가 까봐도 **암호문**입니다.

```
프록시 입장:
  ┌──────────────────────────────────────────┐
  │ TCP 8443                                  │  ← L4 정보는 보임
  │ ┌──────────────────────────────────────┐ │
  │ │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │  ← L7 페이로드는
  │ │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │     암호문 (의미 모름)
  │ └──────────────────────────────────────┘ │
  └──────────────────────────────────────────┘
```

내용을 못 보니 "무슨 요청인지" 판단 불가 → L7 정책 불가. 보이는 건 **L4 정보(목적지 IP·포트)뿐** → **L4 정책만 가능**.

> [!note] [[ztunnel]] mTLS 트래픽이 정확히 이 경우
> 우리 클러스터의 노드 간 트래픽은 `cilium-ztunnel-secrets` CA로 발급된 인증서로 mTLS 암호화됩니다. 그 암호화 구간은 외부 관찰자(다른 프록시·IDS)에게는 L4 정보만 보입니다. L7 검사를 하려면 **복호화 지점 안쪽**에서 해야 함.

### (3) L7 구분이 애초에 의미 없는 통신

```
- DB 커넥션 풀:  "DB에 붙는다/안 붙는다"가 전부. 세부 경로 개념 없음
- Redis:        키-값 명령 단위라 HTTP식 "/path" 모델이 안 맞음
- 단순 TCP 스트림: 구조 자체가 없음 (raw bytes)
- UDP 게임 패킷:  애플리케이션마다 형식이 제각각
```

이런 건 **"누가(IP/identity) 어디(포트)에 붙느냐"**만 통제하면 충분 → L4가 더 자연스럽고 적합.

### (4) 성능 — 굳이 까볼 필요 없을 때

L7 검사는 패킷 페이로드를 파싱하는 비용이 듭니다. DB처럼 대량·고빈도 연결인데 L7로 얻을 게 없다면, **L4로 가볍게 거르는 게 효율적**입니다.

```
처리 비용:
  L4 정책 검사: 포트 비교 1번 (eBPF 해시 조회 ≈ 수십 나노초)
  L7 정책 검사: TCP 재조립 + HTTP 파싱 + 룰 매칭 (수 마이크로초 ~)
                  ↑ 수십 배 비쌈
```

## 정리 — "어느 트래픽이 어느 정책 가능?"

| 트래픽 | L4 가능 | L7 가능 | 이유 |
|---|---|---|---|
| HTTP/1.1, HTTP/2 | ✅ | ✅ | 표준 텍스트, 파서 풍부 |
| [[gRPC]] | ✅ | ✅ | HTTP/2 기반 (정정한 부분) |
| [[Kafka]] | ✅ | ✅ | Cilium에 전용 파서 있음 (토픽 단위 정책) |
| [[DNS]] | ✅ | ✅ | Cilium에 전용 파서 있음 (도메인 단위) |
| PostgreSQL / MySQL | ✅ | ❌ | 전용 바이너리 프로토콜, 파서 없음 |
| [[Redis]] | ✅ | ❌ | RESP 프로토콜, 일반 CNI에 파서 없음 |
| TLS/mTLS 암호화 트래픽 | ✅ | ❌ | 페이로드가 암호문이라 못 읽음 |
| 일반 TCP/UDP 스트림 | ✅ | ❌ | 구조 자체가 없음 |

## [[Cilium]] 맥락에서의 의미

대부분 [[CNI]] ([[Calico]], [[Flannel]] 등)는 **L3/L4 정책까지만** 됩니다. iptables 기반이라 HTTP 내용을 들여다보기 어렵거든요.

[[Cilium]]은 [[eBPF]]로 패킷 내용을 커널에서 파싱할 수 있어서 L7 정책이 추가로 가능합니다:

```
Calico:  L3/L4 정책 (IP, 포트까지)
Cilium:  L3/L4 + L7 정책 (HTTP 경로·메서드, gRPC, Kafka 토픽, DNS 도메인까지)
```

> [!tip] "L7 정책 = Cilium 차별점"의 정확한 의미
> Cilium만 L7이 되고 다른 건 L4도 안 된다는 뜻이 **아닙니다**. L4 정책은 모든 CNI의 기본이고, **거기에 더해 L7까지 된다**는 게 차별점입니다.

## 한 줄 요약

> L4 정책(IP·포트)은 모든 CNI의 기본이고, L7 정책은 그 위에 얹는 "용건 단위 정밀 검문"입니다. L7이 좋은 이유는 같은 포트로 들어오는 정상 요청과 위험 요청을 가를 수 있어서. 그런데 L7은 **프록시가 그 프로토콜을 평문으로 읽고 이해할 수 있을 때만** 성립합니다. DB(전용 바이너리)·암호화 트래픽·구조 없는 스트림은 내용을 못 읽으니 **L4가 "더 적합"한 게 아니라 "유일하게 가능한 선택"**인 경우가 많습니다. 실무는 L4로 굵게, L7로 정밀하게 — 겹쳐 씁니다.

---

## English

> [!tip] TL;DR
> Network policy is **L3/L4 (IP/port) by default** — every [[CNI]] supports it. The problem is L4 is "port open or closed," so it can't distinguish a benign `GET` from a destructive `DELETE` arriving on the same 8080. **L7 policy** parses payload contents and slices by intent. But L7 only works when an L7 proxy like [[Cilium]]/[[Envoy]] **can read and understand that protocol in plaintext**. Databases like [[PostgreSQL]]/[[MySQL]]/[[Redis]] use proprietary binary wire protocols (no parser exists); [[mTLS]]-encrypted traffic is ciphertext to onlookers. For these, **L4 isn't "more appropriate" — it's the only option available.** In practice you layer both: L4 for coarse filtering, L7 for precision.

### First: network layers and what each can see

```
L7 (Application)   HTTP path/method/headers   "GET /api/users"
L6 (Presentation)  (TLS lives here too)
L5 (Session)
L4 (Transport)     port/protocol              "TCP 8080"
L3 (Network)       IP addresses               "10.0.0.5 → 10.0.0.8"
L2 (Data Link)     MAC addresses
L1 (Physical)      electrical signals / cable
```

Higher = closer to **content**; lower = closer to **delivery path**. Network policy is **L3/L4-based by default**, and **L7 sits on top** — anchor this and the rest stops being confusing.

### The same conversation expressed at each layer

Scenario: "Control frontend → backend API traffic."

#### L3 policy (by IP)

```yaml
from:
  ipBlock: 10.0.1.0/24
action: allow
```
→ "Only people from this neighborhood may enter."

#### L4 policy (by port/protocol) — most common

```yaml
ports:
  - port: 8080
    protocol: TCP
action: allow
```
→ "Come in only through the main gate (8080); other doors are locked."

#### L7 policy (by HTTP contents)

```yaml
http:
  - method: GET
    path: /api/products
    action: allow
  - method: DELETE
    path: /api/users/*
    action: deny
  - headers:
      Authorization: "Bearer .*"
    action: allow
```
→ "Yes you can enter, but I'll check **what you're here for** and let you through or block you."

### Why L7 is good — the hard limit of L4

L4's intrinsic limit: **a port is either open or closed.**

```
L4's world: TCP 8080 = OPEN ✅  or  CLOSED ❌
            → once open, ALL requests on that port pass
```

The problem: **benign and dangerous requests share the same port:**

```
TCP 8080 (all on the same port)
   ├─ GET    /api/products        ← normal read (want to allow)
   ├─ GET    /api/users           ← PII read (needs auth)
   ├─ POST   /api/orders          ← order creation (needs auth)
   └─ DELETE /api/users/{id}      ← account deletion (admin only)
```

> [!warning] L4 is all-or-nothing
> **L4 cannot distinguish these.** Open 8080 and all four pass; close it and all four are blocked. To carve "allow reads, deny deletes" inside one port you **must** use L7.

#### Analogy — the guard's field of view

| | L4 guard | L7 guard |
|---|---|---|
| Location | Building front gate | Office doorway |
| Checks | Badge (port) presence | Badge + **what you're here for** |
| Limit | Once inside, anything goes | Controls each action (per request) |

[[Cilium]] uses [[eBPF]] to parse packet contents in-kernel, so L7 policy is possible. iptables-based [[Calico]] can only manage L3/L4 — peering into HTTP contents through iptables chains is impractical.

### Then L4 is useless? — no, you stack both (defense in depth)

> [!note] L4 policy doesn't disappear
> You don't use L7 *instead of* L4; you stack L7 *on top of* L4.

```
1st (L3/L4): "Only from this neighborhood (IP), only through 8080"   ← coarse, fast
        ▼ surviving packets only
2nd (L7):    "Of those, only GET — block DELETE"                     ← precise, expensive
```

Why:

| Reason | Explanation |
|---|---|
| **Performance** | L7 inspection requires payload parsing, which is expensive. Filter 99% with L4 first, then inspect survivors with L7. |
| **Coverage** | Not all traffic is HTTP — DBs, encrypted streams, raw TCP simply can't do L7 (details below). |
| **Defense in depth** | If one layer is misconfigured, the other still catches. L4 is the first-line backstop for any L7 mistake. |

### What "L4 is more appropriate" actually means

> [!warning] Correction
> Earlier I lumped "DB and gRPC" together as L4-appropriate — that's inaccurate. **[[gRPC]] in fact does support L7 policy** (it rides on [[HTTP/2]], so `POST /package.Service/Method` is controllable). The traffic that truly forces L4 is **traffic whose contents the proxy cannot read**.

For L7 policy to be viable, two things must hold:

1. **The proxy must know how to parse the protocol** ([[Cilium]] ships parsers for HTTP, gRPC, Kafka, DNS).
2. **The traffic must be in plaintext**, or the proxy must be able to decrypt it.

If either fails, L7 is fundamentally impossible — and **only L4** remains.

### Four cases where only L4 works (= L4 is "appropriate")

#### (1) Binary protocols the proxy doesn't understand — databases are the canonical case

[[PostgreSQL]], [[MySQL]], [[Redis]] each speak their own **proprietary binary wire protocol**. There's no standardized "path/method" structure like HTTP.

```
HTTP:
  GET /api/users HTTP/1.1\r\n
  Host: api.example.com\r\n
  ← text, well-defined structure, easy to parse

PostgreSQL wire protocol:
  0x51 0x00 0x00 0x00 0x1a 0x53 0x45 0x4c ...
  ← message-type byte + length + payload (binary, DB-specific)

Redis RESP:
  *3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n
  ← its own text protocol, but not HTTP
```

[[Cilium]]'s L7 parsers cover HTTP/gRPC/Kafka/DNS. **It cannot parse PostgreSQL queries to say "allow `SELECT`, deny `DROP TABLE`"** — no parser exists. The practical ceiling for DB traffic is L4: "this Pod is allowed to talk to port 5432":

```
# Reality of DB-traffic policy
from:
  podSelector:
    app: api-server         ← "who" (L3/identity)
to:
  ports:
    - port: 5432            ← "where" (L4)
      protocol: TCP
# "what is being queried" is unknowable → no policy at that level
```

> [!example] Dedicated DB proxies exist for SQL-level policy
> If you genuinely need per-SQL-statement policy, run a DB-specific gateway like [[ProxySQL]] or [[Vitess]]. Generic [[service mesh]]/[[CNI]] layers can't do it.

#### (2) Encrypted traffic — contents invisible

[[TLS]]/[[mTLS]]-encrypted traffic looks like **ciphertext** even to a proxy in the data path.

```
What the proxy sees:
  ┌──────────────────────────────────────────┐
  │ TCP 8443                                  │  ← L4 info visible
  │ ┌──────────────────────────────────────┐ │
  │ │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │  ← L7 payload is
  │ │ ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│ │     ciphertext (opaque)
  │ └──────────────────────────────────────┘ │
  └──────────────────────────────────────────┘
```

You can't tell "what request this is" → no L7. What's visible is **L4 only (destination IP/port)** → **L4 policy only**.

> [!note] [[ztunnel]] mTLS traffic is exactly this case
> Cross-node traffic in our cluster is mTLS-encrypted with certs issued by the `cilium-ztunnel-secrets` CA. Inside that encrypted segment, outside observers (other proxies, IDS) only see L4. L7 inspection has to happen **inside the decryption boundary**.

#### (3) Communication where L7 distinctions are inherently meaningless

```
- DB connection pools:  "connects to DB / doesn't" is the whole policy
- Redis:                key/command-based, doesn't fit an HTTP "/path" model
- Raw TCP streams:      no structure (raw bytes)
- UDP game packets:     application-specific formats, no standard
```

For these, controlling **"who (IP/identity) connects to where (port)"** is enough — L4 is the natural fit.

#### (4) Performance — when there's nothing to gain from looking inside

L7 inspection costs payload parsing. For high-volume, high-frequency connections like DB pools where L7 gives you nothing, L4 is far more efficient:

```
Cost per check:
  L4: single port compare (eBPF hashmap lookup ≈ tens of nanoseconds)
  L7: TCP reassembly + HTTP parse + rule match (microseconds)
                  ↑ tens of times more expensive
```

### Summary table — what policy is possible for what traffic

| Traffic | L4? | L7? | Why |
|---|---|---|---|
| HTTP/1.1, HTTP/2 | ✅ | ✅ | Standard text, mature parsers |
| [[gRPC]] | ✅ | ✅ | Runs on HTTP/2 (the correction) |
| [[Kafka]] | ✅ | ✅ | Cilium ships a parser (topic-level policy) |
| [[DNS]] | ✅ | ✅ | Cilium ships a parser (domain-level policy) |
| PostgreSQL / MySQL | ✅ | ❌ | Proprietary binary; no parser |
| [[Redis]] | ✅ | ❌ | RESP protocol; no parser in mainstream CNIs |
| TLS/mTLS-encrypted | ✅ | ❌ | Payload is ciphertext |
| Raw TCP/UDP streams | ✅ | ❌ | No structure |

### What this means in the [[Cilium]] context

Most [[CNI]]s ([[Calico]], [[Flannel]], ...) only support **L3/L4 policy**. Being iptables-based, they can't economically inspect HTTP contents.

[[Cilium]]'s [[eBPF]] lets it parse packet contents in the kernel, adding L7 on top:

```
Calico:  L3/L4 policy (IPs, ports)
Cilium:  L3/L4 + L7 policy (HTTP path/method, gRPC methods, Kafka topics, DNS names)
```

> [!tip] What "L7 = Cilium's differentiator" really means
> It does **not** mean only Cilium can do L4 — every CNI does L4. It means Cilium does L4 **plus** L7 in the same data plane.

### One-line summary

> L4 policy (IP/port) is the baseline every CNI supports; L7 policy is the precision check stacked on top. L7 is valuable because it can separate safe and dangerous requests sharing the same port. But L7 only works **when the proxy can read and understand the protocol in plaintext**. For DBs (proprietary binary), encrypted traffic, and structureless streams, the proxy cannot read the contents — so **L4 isn't "more appropriate," it's the only option available**. In practice: L4 for coarse filtering, L7 for precision, stacked together.
