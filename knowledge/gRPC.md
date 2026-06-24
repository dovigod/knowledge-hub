---
id: 019e8db0-c3f9-741a-9b5d-2acfb8d19e47
name: gRPC
aliases:
  - GRPC
  - Google RPC
  - gRPC
  - gRPC framework
  - grpc
updated_at: '2026-06-03T13:42:05.790Z'
summary: >-
  High-performance RPC framework built on HTTP/2 and Protocol Buffers, designed
  for strict, code-generated service-to-service communication.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
  - 019e8db1-624f-73be-ab60-735be949b701
  - 019e8db3-b83e-766f-9304-a0ab2827ffaa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# gRPC

## Overview

gRPC is a high-performance Remote Procedure Call framework that delivers fast, strict, and automated service-to-service communication over [[HTTP/2]] using [[Protocol Buffers]] as its interface definition language. It is not simply a "better [[REST]]" — it is a high-performance RPC system purpose-built for internal communication between services.

> [!note] Core Identity
> gRPC = HTTP/2 transport + Protocol Buffers serialization + generated client/server stubs. Strict contracts, not loose JSON.

> [!info] gRPC vs HTTPS is the wrong framing
> gRPC also runs over **HTTPS** (TLS + HTTP/2). The real comparison is **gRPC (HTTP/2 + Protobuf)** vs **[[REST]] (HTTP/1.1 + [[JSON]])** — every speed and reliability claim below traces back to those two axes.

## Notes

> [!tip] Why use it
> - Built on [[HTTP/2]] (multiplexing, header compression, binary framing)
> - Uses [[Protocol Buffers]] for compact, typed payloads
> - Strong performance, type safety, bidirectional streaming, automatic code generation
> - Well suited for internal communication in a [[Microservices]] architecture

> [!warning] Trade-offs
> - Poor browser friendliness (needs [[gRPC-Web]] proxy)
> - Harder to debug than plain [[REST]] / [[JSON]] over HTTP — binary payloads need `grpcurl` instead of `curl`
> - Higher initial complexity and tooling burden

> [!example] When it fits
> Internal east-west traffic between services where latency, schema rigor, and streaming matter more than human-readable payloads or browser reach.

## Why It's Fast

### 1. Protobuf binary serialization

A service contract is defined in a `.proto` file, then client and server code is **auto-generated**:

```protobuf
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  int64 id = 1;        // numeric tag = wire-format key
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

The same payload on the wire:

```
JSON:     {"id":42,"name":"justin","email":"j@x.io"}  → 43 bytes
Protobuf: 0x08 0x2A 0x12 0x06 justin 0x1A 0x06 ...    → ~20 bytes
```

| | JSON | Protobuf |
|---|---|---|
| Field names | Sent as strings every time | 1-byte numeric tags |
| Numbers | Text (`"1234567"` = 7 bytes) | varint (~3 bytes) |
| Parsing | Tokenize text + infer types | Schema known at compile time |

**Parsing cost is the bigger win.** JSON parsing burns CPU branching on `{` vs `"` and converting numbers; Protobuf knows "tag 2 = name, string" ahead of time and effectively memcpys. At high request rates this dominates.

### 2. [[HTTP/2]] over HTTP/1.1

```
HTTP/1.1: one request per connection, in order
  conn1: [req1 ──── res1] [req2 ──── res2]     ← head-of-line blocking

HTTP/2: many streams multiplexed on one connection
  conn1: [req1▓▓░░res1░▓]
         [req2░▓▓res2▓░░]
         [req3▓░░▓res3▓▓]
```

- **Multiplexing** — one persistent connection between services; no repeated TCP + TLS handshakes (2–3 round trips each)
- **HPACK header compression** — repeated headers become table references
- **Binary framing** — no text parsing
- **Streaming** — see below

### 3. Streaming built-in

REST is strictly 1:1 request/response. gRPC defines four patterns:

```protobuf
rpc GetUser(Req) returns (Res);                    // unary (like REST)
rpc ListUsers(Req) returns (stream Res);           // server streaming
rpc UploadLogs(stream Req) returns (Res);          // client streaming
rpc Chat(stream Req) returns (stream Res);         // bidirectional
```

Real-time notifications, large uploads, or chat — handled in the same framework, same connection. No polling, no separate WebSocket layer.

## Why It's Reliable

> [!tip] Strong-typed contracts
> `.proto` is the single source of truth and code is generated from it, so contract violations become **compile errors, not runtime undefineds**. Numbered field tags make backward-compatible evolution explicit: adding a new field is safe; reusing a tag is forbidden.

> [!note] Deadline propagation
> When service A calls B calls C, the remaining time budget flows automatically:
> ```
> A ──(deadline: 200ms)──▶ B ──(remaining: 130ms)──▶ C
> ```
> B and C don't keep working on requests A has already abandoned — a class of waste that's common in [[REST]].

> [!example] Batteries included
> Retry policies, health-check protocol, client-side load balancing, and precise status codes (`UNAVAILABLE`, `DEADLINE_EXCEEDED`, …) ship in the framework. REST teams typically assemble these by hand.

## When NOT to Use gRPC

| Situation | Pick |
|---|---|
| Microservices internal (server ↔ server) | **gRPC** — all strengths apply |
| Browser → server | **[[REST]] / [[JSON]]** — browsers have no native gRPC; [[gRPC-Web]] needs a proxy |
| Public API (external developers) | **REST** — `curl`-friendly, dominant ecosystem and docs |
| Debug ergonomics matter | REST — gRPC payloads are binary; need `grpcurl` |
| Real-time bidirectional | gRPC streaming (or WebSocket) |

## Comparison: HTTP vs gRPC vs MQ

> [!info] Communication style at a glance
> - **[[HTTP]]** — synchronous, request/response; best for external APIs and browser-facing traffic.
> - **gRPC** — synchronous, high-performance RPC; best for internal service-to-service calls where latency and schema rigor matter.
> - **[[Message Queue|MQ]]** ([[Kafka]], [[RabbitMQ]], [[SQS]]) — asynchronous event delivery; decouples producers from consumers and absorbs load spikes.

> [!tip] They are complements, not substitutes
> Large real-world systems typically use all three together: [[HTTP]] at the edge, gRPC between internal services, and an [[Message Queue|MQ]] for asynchronous events and decoupling.

## Summary

> [!note] One-line takeaway
> **Fast** because Protobuf shrinks payloads and skips text parsing × HTTP/2 reuses connections and multiplexes × streaming removes polling.
> **Reliable** because generated strong-typed contracts catch API drift at compile time, and deadlines / retries / health checks are framework-native.

Related: [[REST]], [[HTTP]], [[HTTP/2]], [[HTTPS]], [[TLS]], [[Protocol Buffers]], [[JSON]], [[Microservices]], [[RPC]], [[gRPC-Web]], [[WebSocket]], [[Message Queue]], [[Kafka]], [[RabbitMQ]], [[SQS]]

---

## 한국어

### 개요

gRPC는 [[HTTP/2]] 위에서 [[Protocol Buffers]]를 인터페이스 정의 언어로 사용하여, 빠르고 엄격하며 자동화된 서비스 간 통신을 제공하는 고성능 RPC 프레임워크다. 단순히 "더 좋은 [[REST]]"가 아니라, 서비스 간 내부 통신을 위해 설계된 고성능 RPC 시스템이다.

> [!note] 정체성
> gRPC = HTTP/2 전송 + Protocol Buffers 직렬화 + 자동 생성된 클라이언트/서버 스텁. 느슨한 JSON이 아니라 엄격한 계약 기반.

> [!info] "gRPC vs HTTPS"는 틀린 비교
> gRPC 역시 **HTTPS**(TLS + HTTP/2) 위에서 돈다. 실제 비교는 **gRPC (HTTP/2 + Protobuf)** vs **[[REST]] (HTTP/1.1 + [[JSON]])** 이며, 아래의 모든 속도·안정성 주장은 이 두 축에서 나온다.

### 노트

> [!tip] 왜 쓰는가
> - [[HTTP/2]] 기반 (멀티플렉싱, 헤더 압축, 바이너리 프레이밍)
> - [[Protocol Buffers]]로 작고 타입이 명확한 페이로드
> - 성능, 타입 안정성, 양방향 스트리밍, 코드 자동 생성에 강점
> - [[Microservices]] 내부 통신에 적합

> [!warning] 단점
> - 브라우저 친화성 부족 ([[gRPC-Web]] 프록시 필요)
> - 평범한 [[REST]] / [[JSON]] over HTTP보다 디버깅 어려움 — 바이너리라 `curl` 대신 `grpcurl` 필요
> - 초기 복잡도와 툴링 부담이 큼

> [!example] 적합한 상황
> 사람이 읽는 페이로드나 브라우저 접근성보다 지연 시간, 스키마 엄격성, 스트리밍이 더 중요한 서비스 간 내부 통신.

### 왜 빠른가

#### 1. Protobuf 바이너리 직렬화

서비스 계약을 `.proto` 파일에 먼저 정의하고, 거기서 클라이언트/서버 코드를 **자동 생성**한다:

```protobuf
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  int64 id = 1;        // 숫자 태그 = 와이어 포맷의 키
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

같은 데이터를 와이어로 보낼 때:

```
JSON:     {"id":42,"name":"justin","email":"j@x.io"}  → 43 bytes
Protobuf: 0x08 0x2A 0x12 0x06 justin 0x1A 0x06 ...    → ~20 bytes
```

| | JSON | Protobuf |
|---|---|---|
| 필드 이름 | 매번 문자열로 전송 | 1바이트 숫자 태그 |
| 숫자 | 텍스트 (`"1234567"` = 7바이트) | varint (~3바이트) |
| 파싱 | 텍스트 토크나이징 + 타입 추론 | 스키마를 컴파일 타임에 이미 앎 |

**파싱 비용이 더 큰 차이를 만든다.** JSON 파싱은 `{`인지 `"`인지 분기하고 숫자를 변환하는 CPU 작업이지만, Protobuf는 "태그 2번 = name, string"을 이미 알고 있어 사실상 memcpy에 가깝다. 고트래픽 서버에서는 이 차이가 지배적이다.

#### 2. HTTP/1.1 대비 [[HTTP/2]]

```
HTTP/1.1: 연결당 요청 하나씩 순서대로
  conn1: [req1 ──── res1] [req2 ──── res2]     ← Head-of-Line 블로킹

HTTP/2: 하나의 연결에 여러 스트림 다중화
  conn1: [req1▓▓░░res1░▓]
         [req2░▓▓res2▓░░]
         [req3▓░░▓res3▓▓]
```

- **멀티플렉싱** — 서비스 간 영구 연결 하나를 재사용. TCP + TLS 핸드셰이크(왕복 2~3회)를 매번 다시 하지 않음
- **HPACK 헤더 압축** — 반복 헤더를 테이블 참조로 대체
- **바이너리 프레이밍** — 텍스트 파싱 없음
- **스트리밍** — 아래 참조

#### 3. 내장 스트리밍

REST는 1:1 요청/응답뿐이지만 gRPC는 4가지 패턴을 정의한다:

```protobuf
rpc GetUser(Req) returns (Res);                    // 단항 (REST와 동일)
rpc ListUsers(Req) returns (stream Res);           // 서버 스트리밍
rpc UploadLogs(stream Req) returns (Res);          // 클라이언트 스트리밍
rpc Chat(stream Req) returns (stream Res);         // 양방향
```

실시간 알림, 대용량 업로드, 채팅을 같은 프레임워크·같은 연결에서 처리. 폴링과 별도 WebSocket 레이어가 사라진다.

### 왜 안정적인가

> [!tip] 강타입 계약
> `.proto`가 단일 진실 공급원이고 거기서 코드를 생성하므로 계약 위반이 **런타임 undefined가 아니라 컴파일 에러**가 된다. 필드 번호 덕에 하위 호환 진화 규칙도 명확하다: 새 필드 추가는 안전, 기존 태그 번호 재사용은 금지.

> [!note] 데드라인 전파
> A → B → C로 호출이 이어질 때 남은 시간 예산이 자동으로 흐른다:
> ```
> A ──(deadline: 200ms)──▶ B ──(남은 시간: 130ms)──▶ C
> ```
> A가 이미 포기한 요청을 B와 C가 계속 처리하는 낭비([[REST]]에서 흔한 패턴)가 구조적으로 방지된다.

> [!example] 표준 내장 기능
> 재시도 정책, 헬스체크 프로토콜, 클라이언트 사이드 로드밸런싱, 정확한 의미를 갖는 상태 코드(`UNAVAILABLE`, `DEADLINE_EXCEEDED` 등)가 프레임워크에 들어 있다. REST 팀은 보통 이걸 직접 조립한다.

### gRPC가 답이 아닌 경우

| 상황 | 선택 |
|---|---|
| 마이크로서비스 내부 (서버↔서버) | **gRPC** — 장점이 전부 발휘됨 |
| 브라우저 → 서버 | **[[REST]] / [[JSON]]** — 브라우저는 gRPC 네이티브 지원 없음; [[gRPC-Web]]은 프록시 필요 |
| 공개 API (외부 개발자 대상) | **REST** — `curl`로 찔러볼 수 있고 생태계가 압도적 |
| 디버깅 편의 중요 | REST — gRPC는 바이너리라 `grpcurl` 같은 도구 필요 |
| 실시간 양방향 통신 | gRPC 스트리밍 (또는 WebSocket) |

### 비교: HTTP vs gRPC vs MQ

> [!info] 통신 방식 한눈에 보기
> - **[[HTTP]]** — 동기식 요청/응답. 외부 API와 브라우저 대상 트래픽에 주로 사용.
> - **gRPC** — 동기식 고성능 RPC. 지연 시간과 스키마 엄격성이 중요한 내부 서비스 간 통신에 적합.
> - **[[Message Queue|MQ]]** ([[Kafka]], [[RabbitMQ]], [[SQS]]) — 비동기 이벤트 전달. 생산자와 소비자를 분리하고 부하 급증을 흡수.

> [!tip] 대체재가 아니라 보완재
> 실제 대규모 시스템은 세 가지를 함께 쓴다: 엣지에는 [[HTTP]], 내부 서비스 간에는 gRPC, 비동기 이벤트와 결합도 감소에는 [[Message Queue|MQ]].

### 요약

> [!note] 한 줄 정리
> **빠른 이유** = Protobuf(작은 페이로드 + 싼 파싱) × HTTP/2(연결 재사용 + 멀티플렉싱) × 스트리밍(폴링 제거).
> **안정적인 이유** = 코드 생성 기반 강타입 계약으로 API 불일치가 컴파일 타임에 잡히고, 데드라인·재시도·헬스체크가 프레임워크에 내장.

관련: [[REST]], [[HTTP]], [[HTTP/2]], [[HTTPS]], [[TLS]], [[Protocol Buffers]], [[JSON]], [[Microservices]], [[RPC]], [[gRPC-Web]], [[WebSocket]], [[Message Queue]], [[Kafka]], [[RabbitMQ]], [[SQS]]

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
- [[raw/conversations/019e8db1-624f-73be-ab60-735be949b701|019e8db1-624f-73be-ab60-735be949b701]]
- [[raw/conversations/019e8db3-b83e-766f-9304-a0ab2827ffaa|019e8db3-b83e-766f-9304-a0ab2827ffaa]]
