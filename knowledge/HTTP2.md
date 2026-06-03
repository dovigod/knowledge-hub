---
id: 019e8db0-e52a-74df-9a20-30ca56425c52
name: HTTP/2
aliases:
  - HTTP2
  - h2
  - http2
updated_at: '2026-06-03T13:34:01.002Z'
summary: >-
  Major revision of HTTP that introduces binary framing, multiplexing, header
  compression, and server push over a single TCP connection.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# HTTP/2

## Overview

HTTP/2 is the second major version of the HTTP protocol, replacing HTTP/1.1's text-based, one-request-per-connection model with a binary, multiplexed protocol over a single TCP connection. It is the transport foundation that makes [[gRPC]]'s streaming and low-latency RPC characteristics possible.

> [!note] Why it matters for RPC
> HTTP/2's multiplexing and bidirectional streaming let many concurrent calls share one connection without head-of-line blocking at the HTTP layer — exactly what an RPC framework like [[gRPC]] needs.

## Notes

> [!tip] Key features
> - Binary framing instead of text
> - Request/response multiplexing on a single connection
> - Header compression (HPACK)
> - Server push (now largely deprecated in browsers)
> - Stream prioritization

> [!warning] Caveats
> - Still subject to TCP-level head-of-line blocking (addressed by [[HTTP/3]] / [[QUIC]])
> - Some intermediaries (proxies, load balancers) need explicit HTTP/2 support
> - Browser HTTP/2 typically requires TLS in practice

Used as the transport for [[gRPC]] and as a general performance upgrade over [[HTTP/1.1]] for web traffic.

---

## 한국어

### 개요

HTTP/2는 HTTP 프로토콜의 두 번째 메이저 버전으로, HTTP/1.1의 텍스트 기반·연결당 단일 요청 모델을 단일 TCP 연결 위의 바이너리 멀티플렉스 프로토콜로 대체한다. [[gRPC]]의 스트리밍과 낮은 지연 RPC 특성을 가능케 하는 전송 계층 기반이다.

> [!note] RPC에서 왜 중요한가
> HTTP/2의 멀티플렉싱과 양방향 스트리밍 덕분에 여러 동시 호출이 하나의 연결을 공유하면서도 HTTP 계층에서 헤드 오브 라인 블로킹을 겪지 않는다. [[gRPC]] 같은 RPC 프레임워크가 정확히 필요로 하는 특성이다.

### 노트

> [!tip] 주요 특징
> - 텍스트가 아닌 바이너리 프레이밍
> - 단일 연결 위에서 요청/응답 멀티플렉���
> - 헤더 압축 (HPACK)
> - 서버 푸시 (브라우저에서는 대부분 폐기 수순)
> - 스트림 우선순위

> [!warning] 한계
> - TCP 계층의 헤드 오브 라인 블로킹은 여전 ([[HTTP/3]] / [[QUIC]]가 이를 해결)
> - 일부 중간 장비(프록시, 로드 밸런서)는 명시적 HTTP/2 지원 필요
> - 브라우저에서는 실질적으로 TLS 필수

[[gRPC]]의 전송 계층으로 사용되며, 웹 트래픽에서는 [[HTTP/1.1]] 대비 일반적 성능 업그레이드로 활용된다.

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
