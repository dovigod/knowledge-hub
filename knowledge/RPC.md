---
id: 019e8db0-f49f-753b-9478-79fd9b8e9f57
name: RPC
aliases:
  - RPC
  - Remote Procedure Call
  - remote procedure call
  - remote-procedure-call
updated_at: '2026-06-03T13:44:44.904Z'
summary: >-
  Remote Procedure Call — a communication paradigm where a program invokes a
  procedure on another address space (typically another machine) as if it were a
  local function.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
  - 019e8db3-b83e-766f-9304-a0ab2827ffaa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# RPC

## Overview

RPC (Remote Procedure Call) is a communication paradigm in which a client invokes a procedure that runs in a different address space — usually on another machine — as if it were a local function call. The framework hides the underlying transport, serialization, and error handling so that distributed calls look like ordinary method calls. [[gRPC]] is one modern, high-performance implementation of this paradigm.

> [!note] Core idea
> Make a call across the network feel like a local function call. The wire protocol, serialization format, and error semantics are framework concerns, not application concerns.

> [!info] Not "RPC vs HTTPS"
> Modern RPC frameworks like [[gRPC]] also run over HTTPS (TLS + [[HTTP/2]]). The meaningful comparison is **RPC over HTTP/2 + binary (e.g. [[Protobuf]])** vs **[[REST]] over HTTP/1.1 + [[JSON]]** — the speed and reliability gains come from those choices, not from skipping TLS.

## Notes

> [!tip] Why RPC-over-binary is faster than REST-over-JSON
> - **Compact wire format.** [[Protobuf]] sends numeric field tags + varint-encoded integers instead of repeated field-name strings; a payload that is ~43 bytes as JSON can be ~20 bytes as Protobuf.
> - **Cheap parsing.** JSON parsing is text tokenization + type inference at runtime. Protobuf already knows from the schema that "tag 2 = name, string," so decoding is closer to a memory copy. At high QPS this CPU cost is significant.
> - **Connection reuse via [[HTTP/2]] multiplexing.** Many concurrent streams share one TCP+TLS connection — no per-request handshake, no HTTP/1.1 head-of-line blocking, plus HPACK header compression.
> - **Streaming patterns.** Unary, server-streaming, client-streaming, and bidirectional streaming are built in, so realtime traffic doesn't need a parallel [[WebSocket]] stack or polling.

> [!tip] Compared to [[REST]]
> - RPC is action-oriented (call a procedure); REST is resource-oriented (operate on resources via HTTP verbs).
> - RPC frameworks like [[gRPC]] favor strict contracts and binary formats; REST commonly favors [[JSON]] and loose contracts.
> - RPC tends to win for internal service-to-service traffic; REST tends to win for public, browser-facing APIs.

> [!tip] Why RPC-style contracts are more reliable
> - **Code-generated contracts.** The `.proto` (or equivalent IDL) is the single source of truth; client/server stubs are generated, so a renamed or missing field is a **compile-time error**, not a silent `undefined` at runtime.
> - **Deadline propagation.** A → B → C calls carry the remaining budget automatically; B and C don't keep working on a request A has already given up on.
> - **Standardized status codes.** Codes like `UNAVAILABLE`, `DEADLINE_EXCEEDED` carry sharper meaning than HTTP's overloaded 4xx/5xx.
> - **Batteries included.** Retries, health checks, and client-side load balancing are framework-level features, not things each team reassembles.

> [!warning] Watch-outs
> - The "looks like a local call" abstraction hides real network failure modes (timeouts, partial failures, retries).
> - Tight schema coupling between client and server requires disciplined versioning (safe to add new fields; never reuse old tag numbers).
> - Binary wire formats are harder to debug — you need tooling like `grpcurl` rather than `curl`.
> - Browsers don't speak [[gRPC]] natively; `grpc-web` needs a proxy and has limits. For public APIs, REST is usually the pragmatic choice.

## Examples

```protobuf
// user.proto — the contract is the source of truth
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  int64 id = 1;        // field tag — used on the wire instead of the name
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

Generated stubs let the client write what looks like a local call:

```typescript
const user = await client.getUser({ id: 42 });  // network call, but typed like a function
```

Modern RPC stacks include [[gRPC]], Thrift, and JSON-RPC. Used heavily in [[Microservices]] architectures.

---

## 한국어

### 개요

RPC(Remote Procedure Call, 원격 프로시저 호출)는 클라이언트가 다른 주소 공간 — 보통 다른 머신 — 에서 실행되는 프로시저를 마치 로컬 함수처럼 호출하는 통신 패러다임이다. 프레임워크가 전송, 직렬화, 오류 처리를 숨겨서 분산 호출을 일반 메서드 호출처럼 보이게 만든다. [[gRPC]]는 이 패러다임의 현대적이고 고성능인 구현 중 하나다.

> [!note] 핵심 아이디어
> 네트워크를 가로지르는 호출을 로컬 함수 호출처럼 느끼게 하자. 와이어 프로토콜, 직렬화 포맷, 오류 의미론은 프레임워크의 관심사이지 애플리케이션의 관심사가 아니다.

> [!info] "RPC vs HTTPS"가 아니다
> [[gRPC]] 같은 현대 RPC 프레임워크도 HTTPS(TLS + [[HTTP/2]]) 위에서 동작한다. 의미 있는 비교는 **HTTP/2 + 바이너리(예: [[Protobuf]]) 기반 RPC** 와 **HTTP/1.1 + [[JSON]] 기반 [[REST]]** 의 비교다 — 속도·안정성 이득은 TLS를 빼서가 아니라 이 선택들에서 나온다.

### 노트

> [!tip] 바이너리 기반 RPC가 JSON 기반 REST보다 빠른 이유
> - **작은 와이어 포맷.** [[Protobuf]]는 필드명을 매번 문자열로 보내는 대신 숫자 태그와 varint 인코딩된 정수를 사용한다. JSON으로 약 43바이트인 페이로드가 Protobuf로는 약 20바이트가 된다.
> - **싼 파싱.** JSON 파싱은 런타임에 텍스트 토크나이징과 타입 추론을 수행한다. Protobuf는 스키마에서 "태그 2번 = name, string"임을 이미 알기 때문에 디코딩이 메모리 복사에 가깝다. 고 QPS에서는 이 CPU 비용 차이가 크다.
> - **[[HTTP/2]] 멀티플렉싱 기반 연결 재사용.** 여러 동시 스트림이 TCP+TLS 연결 하나를 공유한다 — 요청마다 핸드셰이크가 없고, HTTP/1.1의 head-of-line blocking도 없으며, HPACK 헤더 압축까지 얹힌다.
> - **스트리밍 패턴.** 단항, 서버 스트리밍, 클라이언트 스트리밍, 양방향 스트리밍이 기본 제공되므로 실시간 트래픽에 별도의 [[WebSocket]] 스택이나 폴링이 필요 없다.

> [!tip] [[REST]]와의 비교
> - RPC는 행위 중심(프로시저 호출), REST는 리소스 중심(HTTP 동사로 리소스 조작).
> - [[gRPC]] 같은 RPC 프레임워크는 엄격한 계약과 바이너리 포맷을 선호하고, REST는 [[JSON]]과 느슨한 계약을 선호하는 경우가 많다.
> - RPC는 서비스 간 내부 트래픽에, REST는 공개·브라우저 대상 API에 강점을 갖는 경향.

> [!tip] RPC 스타일 계약이 더 안정적인 이유
> - **코드 생성 기반 계약.** `.proto`(또는 동등한 IDL)가 단일 진실 공급원이고 클라이언트/서버 스텁이 거기서 생성되기 때문에, 필드명을 잘못 쓰거나 누락하면 런타임의 조용한 `undefined`가 아니라 **컴파일 에러**가 된다.
> - **데드라인 전파.** A → B → C로 이어지는 호출이 남은 시간 예산을 자동으로 전달한다 — A가 이미 포기한 요청을 B와 C가 계속 처리하는 낭비가 없다.
> - **표준화된 상태 코드.** `UNAVAILABLE`, `DEADLINE_EXCEEDED` 같은 코드가 HTTP의 모호한 4xx/5xx보다 의미가 정확하다.
> - **내장 기능들.** 재시도, 헬스체크, 클라이언트 사이드 로드밸런싱이 프레임워크 수준 기능으로 들어있어 팀마다 새로 조립할 필요가 없다.

> [!warning] 주의점
> - "로컬 호출처럼 보인다"는 추상화는 실제 네트워크 실패 모드(타임아웃, 부분 실패, 재시도)를 가린다.
> - 클라이언트와 서버 사이의 강한 스키마 결합은 규율 있는 버저닝을 요구한다 (새 필드 추가는 안전, 기존 태그 번호 재사용은 금지).
> - 바이너리 와이어 포맷은 디버깅이 어렵다 — `curl` 대신 `grpcurl` 같은 도구가 필요하다.
> - 브라우저는 [[gRPC]]를 네이티브로 지원하지 않는다; `grpc-web`은 프록시가 필요하고 ��약이 있다. 공개 API에서는 보통 REST가 실용적이다.

### 예시

```protobuf
// user.proto — 계약이 곧 진실의 출처
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

message GetUserRequest {
  int64 id = 1;        // 필드 태그 — 와이어 위에서 이름 대신 사용됨
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

생성된 스텁 덕분에 클라이언트는 로컬 함수처럼 호출할 수 있다:

```typescript
const user = await client.getUser({ id: 42 });  // 네트워크 호출인데 함수처럼 타입까지 잡힘
```

현대 RPC 스택에는 [[gRPC]], Thrift, JSON-RPC 등이 있다. [[Microservices]] 아키텍처에서 광범위하게 쓰인다.

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
- [[raw/conversations/019e8db3-b83e-766f-9304-a0ab2827ffaa|019e8db3-b83e-766f-9304-a0ab2827ffaa]]
