---
id: 019e8db0-e52a-74df-9a20-30ca56425c52
name: HTTP/2
aliases:
  - HTTP/2
  - HTTP2
  - h2
  - http2
updated_at: '2026-06-03T13:43:22.361Z'
summary: >-
  Major revision of HTTP that introduces binary framing, multiplexing, header
  compression, and server push over a single TCP connection.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
  - 019e8db3-b83e-766f-9304-a0ab2827ffaa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# HTTP/2

## Overview

HTTP/2 is the second major version of the HTTP protocol, replacing HTTP/1.1's text-based, one-request-per-connection model with a binary, multiplexed protocol over a single TCP connection. It is the transport foundation that makes [[gRPC]]'s streaming and low-latency RPC characteristics possible.

> [!note] Why it matters for RPC
> HTTP/2's multiplexing and bidirectional streaming let many concurrent calls share one connection without head-of-line blocking at the HTTP layer — exactly what an RPC framework like [[gRPC]] needs. Long-lived connections also amortize away the TCP + TLS handshake cost (2–3 round trips) that HTTP/1.1 would otherwise pay repeatedly.

## Notes

> [!tip] Key features
> - Binary framing instead of text (no tokenizing cost)
> - Request/response multiplexing on a single connection
> - Header compression ([[HPACK]]) — repeated headers replaced with table references
> - Server push (now largely deprecated in browsers)
> - Stream prioritization

> [!example] Multiplexing vs HTTP/1.1
> ```
> HTTP/1.1: one request per connection, in order
>   conn1: [req1 ──── res1] [req2 ──── res2]
>   (browsers historically opened ~6 parallel connections to work around this)
>
> HTTP/2: many streams interleaved on one connection
>   conn1: [req1▓▓░░res1░▓]
>          [req2░▓▓res2▓░░]
>          [req3▓░░▓res3▓▓]
> ```

> [!warning] Caveats
> - Still subject to TCP-level head-of-line blocking (addressed by [[HTTP/3]] / [[QUIC]])
> - Some intermediaries (proxies, load balancers) need explicit HTTP/2 support
> - Browser HTTP/2 typically requires TLS in practice
> - Binary wire format is harder to inspect than HTTP/1.1 text (need tools like `grpcurl` for [[gRPC]])

Used as the transport for [[gRPC]] and as a general performance upgrade over [[HTTP/1.1]] for web traffic. In microservice-to-microservice traffic the connection-reuse win is especially large, since the same client/server pair exchange many requests over hours.

---

## 한국어

### 개요

HTTP/2는 HTTP 프로토콜의 두 번째 메이저 버전으로, HTTP/1.1의 텍스트 기반·연결당 단일 요청 모델을 단일 TCP 연결 위의 바이너리 멀티플렉스 프로토콜로 대체한다. [[gRPC]]의 스트리밍과 낮은 지연 RPC 특성을 가능케 하는 전송 계층 기반이다.

> [!note] RPC에서 왜 중요한가
> HTTP/2의 멀티플렉싱과 양방향 스트리밍 덕분에 여러 동시 호출이 하나의 연결을 공유하면서도 HTTP 계층에서 헤드 오브 라인 블로킹을 겪지 않는다. [[gRPC]] 같은 RPC 프레임워크가 정확히 필요로 하는 특성이다. 장수명 연결은 HTTP/1.1이 반복적으로 치러야 했던 TCP + TLS 핸드셰이크 비용(왕복 2~3회)도 분할 상환해 준다.

### 노트

> [!tip] 주요 특징
> - 텍스트가 아닌 바이너리 프레이밍 (토크나이징 비용 없음)
> - 단일 연결 위에서 요청/응답 멀티플렉싱
> - 헤더 압축 ([[HPACK]]) — 반복되는 헤더를 테이블 참조로 대체
> - 서버 푸시 (브라우저에서는 대부분 폐기 수순)
> - 스트림 우선순위

> [!example] HTTP/1.1과의 멀티플렉싱 비교
> ```
> HTTP/1.1: 연결 하나당 요청 하나씩 순서대로
>   conn1: [req1 ──── res1] [req2 ──── res2]
>   (그래서 브라우저는 역사적으로 연결을 6개씩 열어 우회)
>
> HTTP/2: 단일 연결에서 여러 스트림을 동시 다중화
>   conn1: [req1▓▓░░res1░▓]
>          [req2░▓▓res2▓░░]
>          [req3▓░░▓res3▓▓]
> ```

> [!warning] 한계
> - TCP 계층의 헤드 오브 라인 블로킹은 여전 ([[HTTP/3]] / [[QUIC]]가 이를 해결)
> - 일부 중간 장비(프록시, 로드 밸런서)는 명시적 HTTP/2 지원 필요
> - 브라우저에서는 실질적으로 TLS 필수
> - 바이너리 와이어 포맷은 HTTP/1.1 텍스트보다 들여다보기 어려움 ([[gRPC]]의 경우 `grpcurl` 같은 도구 필요)

[[gRPC]]의 전송 계층으로 사용되며, 웹 트래픽에서는 [[HTTP/1.1]] 대비 일반적 성능 업그레이드로 활용된다. 특히 마이크로서비스 간 통신에서는 같은 클라이언트/서버 쌍이 수 시간에 걸쳐 수많은 요청을 주고받기 때문에 연결 재사용 이득이 특히 크다.

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
- [[raw/conversations/019e8db3-b83e-766f-9304-a0ab2827ffaa|019e8db3-b83e-766f-9304-a0ab2827ffaa]]
