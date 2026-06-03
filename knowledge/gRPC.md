---
id: 019e8db0-c3f9-741a-9b5d-2acfb8d19e47
name: gRPC
aliases:
  - GRPC
  - Google RPC
  - grpc
updated_at: '2026-06-03T13:33:52.505Z'
summary: >-
  High-performance RPC framework built on HTTP/2 and Protocol Buffers, designed
  for strict, code-generated service-to-service communication.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# gRPC

## Overview

gRPC is a high-performance Remote Procedure Call framework that delivers fast, strict, and automated service-to-service communication over [[HTTP/2]] using [[Protocol Buffers]] as its interface definition language. It is not simply a "better [[REST]]" — it is a high-performance RPC system purpose-built for internal communication between services, and its value depends entirely on the problem you are solving.

> [!note] Core Identity
> gRPC = HTTP/2 transport + Protocol Buffers serialization + generated client/server stubs. Strict contracts, not loose JSON.

## Notes

> [!tip] Why use it
> - Built on [[HTTP/2]] (multiplexing, header compression, binary framing)
> - Uses [[Protocol Buffers]] for compact, typed payloads
> - Strong performance, type safety, bidirectional streaming, automatic code generation
> - Well suited for internal communication in a [[Microservices]] architecture

> [!warning] Trade-offs
> - Poor browser friendliness (needs [[gRPC-Web]] proxy)
> - Harder to debug than plain [[REST]] / [[JSON]] over HTTP
> - Higher initial complexity and tooling burden

> [!example] When it fits
> Internal east-west traffic between services where latency, schema rigor, and streaming matter more than human-readable payloads or browser reach.

Related: [[REST]], [[HTTP/2]], [[Protocol Buffers]], [[Microservices]], [[RPC]]

---

## 한국어

### 개요

gRPC는 [[HTTP/2]] 위에서 [[Protocol Buffers]]를 인터페이스 정의 언어로 사용하여, 빠르고 엄격하며 자동화된 서비스 간 통신을 제공하는 고성능 RPC 프레임워크다. 단순히 "더 좋은 [[REST]]"가 아니라, 서비스 간 내부 통신을 위해 설계된 고성능 RPC 시스템이며, 가치는 해결하려는 문제에 따라 달라진다.

> [!note] 정체성
> gRPC = HTTP/2 전송 + Protocol Buffers 직렬화 + 자동 생성된 클라이언트/서버 스텁. 느슨한 JSON이 아니라 엄격한 계약 기반.

### 노트

> [!tip] 왜 쓰는가
> - [[HTTP/2]] 기반 (멀티플렉싱, 헤더 압축, 바이너리 프레이밍)
> - [[Protocol Buffers]]로 작고 타입이 명확한 페이로드
> - 성능, 타입 안정성, 양방향 스트리밍, 코드 자동 생성에 강점
> - [[Microservices]] 내부 통신에 적합

> [!warning] 단점
> - 브라우저 친화성 부족 ([[gRPC-Web]] 프록시 필요)
> - 평범한 [[REST]] / [[JSON]] over HTTP보다 디버깅이 어려움
> - 초기 복잡도와 툴링 부담이 큼

> [!example] 적합한 상황
> 사람이 읽는 페이로드나 브라우저 접근성보다 지연 시간, 스키마 엄격성, 스트리밍이 더 중요한 서비스 간 내부 통신.

관련: [[REST]], [[HTTP/2]], [[Protocol Buffers]], [[Microservices]], [[RPC]]

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
