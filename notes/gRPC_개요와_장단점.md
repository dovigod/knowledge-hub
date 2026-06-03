---
id: 019e8db1-53c3-75f8-976a-ecc4cc135475
title: gRPC 개요와 장단점
topics:
  - grpc
  - rpc
  - http/2
  - 마이크로서비스
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
created_at: '2026-06-03T13:34:29.314Z'
updated_at: '2026-06-03T13:34:29.314Z'
---
## Overview

At its core, gRPC is a framework that provides "fast, strict, and automated RPC communication."
That said, it's not unconditionally good — its value depends on what problem you're trying to solve.

## Why use it

- Based on [[HTTP/2]], uses [[Protocol Buffers]]
- Advantages in performance, type safety, streaming, and code generation
- Suitable for internal communication between [[microservices]]

## Drawbacks

- Lack of browser friendliness
- Difficult debugging
- Increased initial complexity

## Conclusion

It is not a "better [[REST]]" but rather a "high-performance [[RPC]] system for inter-service communication."

---

## 한국어

### 개요

핵심부터 말하면, [[gRPC]]는 "빠르고, 엄격하고, 자동화된 RPC 통신"을 제공하는 프레임워크다.
그렇다고 무조건 좋은 건 아니고, 어떤 문제를 해결하려는지에 따라 가치가 갈린다.

### 왜 쓰는가

- [[HTTP/2]] 기반, [[Protocol Buffers]] 사용
- 성능, 타입 안정성, 스트리밍, 코드 생성 장점
- [[마이크로서비스]] 내부 통신에 적합

### 단점

- 브라우저 친화성 부족
- 디버깅 어려움
- 초기 복잡도 증가

### 결론

'더 좋은 [[REST]]'가 아니라 '서비스 간 통신을 위한 고성능 [[RPC]] 시스템'이다.
