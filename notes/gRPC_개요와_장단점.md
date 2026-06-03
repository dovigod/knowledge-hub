---
id: 019e8db1-53c3-75f8-976a-ecc4cc135475
title: gRPC 개요와 장단점
topics:
  - grpc
  - rpc
  - http/2
  - 마이크로서비스
  - protobuf
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
  - 019e8db3-b83e-766f-9304-a0ab2827ffaa
created_at: '2026-06-03T13:34:29.314Z'
updated_at: '2026-06-03T13:38:47.603Z'
---
## gRPC explanation

First, the comparison target needs to be set correctly. gRPC also **runs on top of HTTPS** (TLS + HTTP/2). So the actual comparison isn't "gRPC vs HTTPS", but:

> **gRPC (HTTP/2 + Protobuf)** vs **REST (HTTP/1.1 + JSON)**

All the reasons for being fast and reliable come from these two differences.

## What gRPC is

A **[[RPC]] (Remote Procedure Call) framework** created by Google. Core idea: treat network calls not as "sending a request to a URL" but as **"calling a remote function."**

```protobuf
// user.proto — define the service contract first
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  int64 id = 1;        // ← this number is the field's "tag" (the key in the wire format)
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

Client/server code is **auto-generated** from this `.proto` file. The client just writes it like a function call:

```typescript
const user = await client.getUser({ id: 42 });  // a network call, but used like a function
```

## Why it's fast — reason 1: [[Protocol Buffers|Protobuf]] binary serialization

If you send the same data with JSON and Protobuf:

```
JSON:     {"id":42,"name":"justin","email":"j@x.io"}     → 43 bytes (text)
Protobuf: 0x08 0x2A 0x12 0x06 justin 0x1A 0x06 j@x.io    → ~20 bytes (binary)
```

Sources of the difference:

| | JSON | Protobuf |
|---|---|---|
| Field names | Sent as strings every time (`"email":`) | 1-byte numeric tag (`3`) |
| Numbers | String `"1234567"` (7 bytes) | varint encoding (~3 bytes) |
| Parsing | Text tokenizing + type inference | Schema already known, decode directly |

**Parsing cost is particularly large.** JSON parsing is CPU work that reads strings while branching on whether something is `{` or `"` and converting numbers, whereas Protobuf already knows at compile time that "tag 2 = name, string", so it essentially copies it straight into memory. On high-traffic servers, serialization/deserialization consumes a significant fraction of CPU, so this difference is substantial.

## Why it's fast — reason 2: [[HTTP/2]]

The problem with HTTP/1.1, which REST usually uses:

```
HTTP/1.1: one request at a time per connection, in order
  conn1: [req1 ──── res1] [req2 ──── res2]     ← next request must wait for previous to finish
  (so browsers work around this by opening 6 connections)

HTTP/2: multiple streams multiplexed on a single connection
  conn1: [req1▓▓░░res1░▓]
         [req2░▓▓res2▓░░]   ← simultaneously on one TCP connection, in any order
         [req3▓░░▓res3▓▓]
```

What gRPC gets from HTTP/2:

- **Multiplexing**: in [[microservices]] communication, one connection is reused continuously. Skip redoing TCP handshake + TLS handshake (2~3 round trips) every time
- **Header compression (HPACK)**: replace repeating headers with table references
- **Binary framing**: no text parsing
- **Streaming**: explained below

## Why it's fast — reason 3: Streaming

REST is request-response 1:1 only, but gRPC has 4 built-in communication patterns:

```protobuf
rpc GetUser(Req) returns (Res);                    // 1. unary (same as REST)
rpc ListUsers(Req) returns (stream Res);           // 2. server streaming
rpc UploadLogs(stream Req) returns (Res);          // 3. client streaming
rpc Chat(stream Req) returns (stream Res);         // 4. bidirectional streaming
```

Things that would require polling or bolting on [[WebSocket]] in REST (real-time notifications, large uploads, chat) are handled within the same connection, same framework. Eliminating polling = eliminating unnecessary round trips.

## Why it's "reliable"

This is the more underrated advantage compared to speed.

**1. Strong-typed contract** — common REST accident:

```typescript
// REST: server renamed a field but client doesn't know → undefined at runtime
const email = response.data.emial;  // compiler doesn't catch the typo either
```

In gRPC, `.proto` is the single source of truth and code is generated from it, so contract violations become **compile errors, not runtime bugs**. Thanks to numbering fields, backward-compatible evolution rules are also clear (adding a new field is safe, reusing an existing tag number is forbidden).

**2. Deadline propagation** — when a call chains through services A→B→C:

```
A ──(deadline: 200ms)──▶ B ──(remaining time auto-propagated: 130ms)──▶ C
```

The waste of B and C continuing to process a request A has already given up on (common in REST) is structurally prevented.

**3. Built-in features** — things you'd assemble yourself in REST are standard: retry policy, health-check protocol, client-side load balancing, standardized status codes (`UNAVAILABLE`, `DEADLINE_EXCEEDED`, etc. — more precise meaning than HTTP's ambiguous 4xx/5xx).

## So is gRPC always the answer? No

| Situation | Choice |
|---|---|
| Microservice internal communication (server↔server) | **gRPC** — all advantages come into play |
| Browser → server | **REST/JSON** — browsers have no native gRPC support (grpc-web requires a proxy + has constraints) |
| Public API (for external developers) | **REST** — can be poked with curl, ecosystem/documentation conventions are overwhelming |
| When debugging convenience matters | REST — gRPC is binary so Wireshark can't read it (needs tools like `grpcurl`) |
| Real-time bidirectional communication | gRPC streaming (or WebSocket) |

## Summary

> Why gRPC is fast = **Protobuf** (small payload + cheap parsing) × **HTTP/2** (connection reuse + multiplexing) × **streaming** (polling eliminated).
> Why it's reliable = **code-generation-based strong-typed contracts** catch API mismatches at compile time, and deadlines/retries/health-checks are built into the framework.
>
> However, this is the story on its intended stage of "communication between internal services"; for browsers/public APIs, REST is still practical.

---

## 한국어

### gRPC 설명

먼저 비교 대상을 정확히 잡아야 합니다. gRPC도 **HTTPS 위에서 돕니다** (TLS + HTTP/2). 그래서 실제 비교는 "gRPC vs HTTPS"가 아니라:

> **gRPC (HTTP/2 + Protobuf)** vs **REST (HTTP/1.1 + JSON)**

입니다. 빠르고 안정적인 이유는 전부 이 두 가지 차이에서 나옵니다.

### gRPC란

Google이 만든 **[[RPC]](Remote Procedure Call) 프레임워크**입니다. 핵심 발상: 네트워크 호출을 "URL에 요청 보내기"가 아니라 **"원격 함수 호출하기"**처럼 다루자.

```protobuf
// user.proto — 서비스 계약을 먼저 정의
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  int64 id = 1;        // ← 이 숫자가 필드의 "태그" (와이어 포맷의 키)
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

이 `.proto` 파일에서 클라이언트/서버 코드를 **자동 생성**합니다. 클라이언트는 그냥 함수 호출하듯 씁니다:

```typescript
const user = await client.getUser({ id: 42 });  // 네트워크 호출인데 함수처럼
```

### 왜 빠른가 — 이유 1: [[Protocol Buffers|Protobuf]] 바이너리 직렬화

JSON과 Protobuf로 같은 데이터를 보내면:

```
JSON:     {"id":42,"name":"justin","email":"j@x.io"}     → 43 bytes (텍스트)
Protobuf: 0x08 0x2A 0x12 0x06 justin 0x1A 0x06 j@x.io    → ~20 bytes (바이너리)
```

차이의 원인:

| | JSON | Protobuf |
|---|---|---|
| 필드 이름 | 매번 문자열로 전송 (`"email":`) | 숫자 태그 1바이트 (`3`) |
| 숫자 | 문자열 `"1234567"` (7바이트) | varint 인코딩 (~3바이트) |
| 파싱 | 텍스트 토크나이징 + 타입 추론 | 스키마를 이미 알므로 바로 디코딩 |

**파싱 비용이 특히 큽니다.** JSON 파싱은 문자열을 읽으며 `{`인지 `"`인지 분기하고 숫자를 변환하는 CPU 작업인데, Protobuf는 "태그 2번 = name, string" 을 컴파일 타임에 이미 알고 있어서 메모리에 거의 그대로 복사합니다. 고트래픽 서버에서 직렬화/역��렬화가 CPU의 상당 부분을 먹기 때문에 이 차이가 실질적입니다.

### 왜 빠른가 — 이유 2: [[HTTP/2]]

REST가 보통 쓰는 HTTP/1.1의 문제:

```
HTTP/1.1: 연결 하나당 요청 하나씩 순서대로
  conn1: [req1 ──── res1] [req2 ──── res2]     ← 앞 요청이 끝나야 다음
  (그래서 브라우저가 연결을 6개씩 열어 우회)

HTTP/2: 연결 하나에 여러 스트림 동시 다중화(multiplexing)
  conn1: [req1▓▓░░res1░▓]
         [req2░▓▓res2▓░░]   ← 한 TCP 연결에서 동시에, 순서 무관
         [req3▓░░▓res3▓▓]
```

gRPC가 HTTP/2에서 얻는 것:

- **멀티플렉싱**: [[마이크로서비스]] 간 통신에서 연결 하나를 계속 재사용. TCP 핸드셰이크 + TLS 핸드셰이크(왕복 2~3회)를 매번 다시 하지 않음
- **헤더 압축(HPACK)**: 반복되는 헤더를 테이블 참조로 대체
- **바이너리 프레이밍**: 텍스트 파싱 없음
- **스트리밍**: 아래에서 설명

### 왜 빠른가 — 이유 3: 스트리밍

REST는 요청-응답 1:1뿐이지만, gRPC는 4가지 통신 패턴이 내장돼 있습니다:

```protobuf
rpc GetUser(Req) returns (Res);                    // 1. 단항 (REST와 동일)
rpc ListUsers(Req) returns (stream Res);           // 2. 서버 스트리밍
rpc UploadLogs(stream Req) returns (Res);          // 3. 클라이언트 스트리밍
rpc Chat(stream Req) returns (stream Res);         // 4. 양방향 스트리밍
```

REST라면 폴링하거나 [[WebSocket]]을 따로 붙여야 할 일(실시간 알림, 대용량 업로드, 채팅)을 같은 연결, 같은 프레임워크 안에서 처리합니다. 폴링 제거 = 불필요한 왕복 제거.

### 왜 "안정적"인가

속도보다 이쪽이 과소평가되는 장점입니다.

**1. 강타입 계약(contract)** — REST의 흔한 사고:

```typescript
// REST: 서버가 필드명을 바꿨는데 클라이언트는 모름 → 런타임에 undefined
const email = response.data.emial;  // 오타도 컴파일러가 못 잡음
```

gRPC는 `.proto`가 단일 진실 공급원이고 거기서 코드를 생성하므로, 계약 위반이 **런타임 버그가 아니라 컴파일 에러**가 됩니다. 필드에 번호를 매기는 방식 덕에 하위 호환 진화 규칙도 명확합니다(새 필드 추가는 안전, 기존 태그 번호 재사용은 금지).

**2. 데드라인 전파** — 서비스 A→B→C로 호출이 이어질 때:

```
A ──(deadline: 200ms)──▶ B ──(남은 시간 자동 전파: 130ms)──▶ C
```

A가 이미 포기한 요청을 B와 C가 계속 처리하는 낭비(REST에서 흔함)가 구조적으로 방지됩니다.

**3. 내장 기능들** — REST에서는 직접 조립해야 하는 것들이 표준으로 들어 있음: 재시도 정책, 헬스체크 프로토콜, 클라이언트 사이드 로드밸런싱, 표준화된 상태 코드(`UNAVAILABLE`, `DEADLINE_EXCEEDED` 등 — HTTP의 모호한 4xx/5xx보다 의미가 정확).

### 그럼 항상 gRPC가 답인가? 아니요

| 상황 | 선택 |
|---|---|
| 마이크로서비스 내부 통신 (서버↔서버) | **gRPC** — 장점이 전부 발휘됨 |
| 브라우저 → 서버 | **REST/JSON** — 브라우저는 gRPC 네이티브 지원 없음 (grpc-web은 프록시 필요 + 제약) |
| 공개 API (외부 개발자 대상) | **REST** — curl로 찔러볼 수 있고, 생태계/문서화 관습이 압도적 |
| 디버깅 편의가 중요 | REST — gRPC는 바이너리라 와이어샤크로 봐도 안 읽힘 (`grpcurl` 같은 도구 필요) |
| 실시간 양방향 통신 | gRPC 스트리밍 (또는 WebSocket) |

### 요약

> gRPC가 빠른 이유 = **Protobuf**(작은 페이로드 + 싼 파싱) × **HTTP/2**(연결 재사용 + 멀티플렉싱) × **스트리밍**(폴링 제거).
> 안정적인 이유 = **코드 생성 기반 강타입 계약**으로 API 불일치가 컴파일 타임에 잡히고, 데드라인/재시도/헬스체크가 프레임워크에 내장되어 있어서.
>
> 단, "내부 서비스 간 통신"이라는 본래 무대에서의 이야기이고, 브라우저/공개 API에서는 여전히 REST가 실용적입니다.
