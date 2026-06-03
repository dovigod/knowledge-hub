---
id: 019e8db0-f49f-753b-9478-79fd9b8e9f57
name: RPC
aliases:
  - Remote Procedure Call
  - remote procedure call
updated_at: '2026-06-03T13:34:04.959Z'
summary: >-
  Remote Procedure Call — a communication paradigm where a program invokes a
  procedure on another address space (typically another machine) as if it were a
  local function.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# RPC

## Overview

RPC (Remote Procedure Call) is a communication paradigm in which a client invokes a procedure that runs in a different address space — usually on another machine — as if it were a local function call. The framework hides the underlying transport, serialization, and error handling so that distributed calls look like ordinary method calls. [[gRPC]] is one modern, high-performance implementation of this paradigm.

> [!note] Core idea
> Make a call across the network feel like a local function call. The wire protocol, serialization format, and error semantics are framework concerns, not application concerns.

## Notes

> [!tip] Compared to [[REST]]
> - RPC is action-oriented (call a procedure); REST is resource-oriented (operate on resources via HTTP verbs).
> - RPC frameworks like [[gRPC]] favor strict contracts and binary formats; REST commonly favors [[JSON]] and loose contracts.
> - RPC tends to win for internal service-to-service traffic; REST tends to win for public, browser-facing APIs.

> [!warning] Watch-outs
> - The "looks like a local call" abstraction hides real network failure modes (timeouts, partial failures, retries).
> - Tight schema coupling between client and server requires disciplined versioning.

Modern RPC stacks include [[gRPC]], Thrift, and JSON-RPC. Used heavily in [[Microservices]] architectures.

---

## 한국어

### 개요

RPC(Remote Procedure Call, 원격 프로시저 호출)는 클라이언트가 다른 주소 공간 — 보통 다른 머신 — 에서 실행되는 프로시저를 마치 로컬 함수처럼 호출하는 통신 패러다임이다. 프레임워크가 전송, 직렬화, 오류 처리를 숨겨서 분산 호출을 일반 메서드 호출처럼 보이게 만든다. [[gRPC]]는 이 패러다임의 현대적이고 고성능인 구현 중 하나다.

> [!note] 핵심 아이디어
> 네트워크를 가로지르는 호출을 로컬 함수 호출처럼 느끼게 하자. 와이어 프로토콜, 직렬화 포맷, 오류 의미론은 프레임워크의 관심사이지 애플리케이션의 관심사가 아니다.

### 노트

> [!tip] [[REST]]와의 비교
> - RPC는 행위 중심(프로시저 호출), REST는 리소스 중심(HTTP 동사로 리소스 조작).
> - [[gRPC]] 같은 RPC 프레임워크는 엄격한 계약과 바이너리 포맷을 선호하고, REST는 [[JSON]]과 느슨한 계약을 선호하는 경우가 많다.
> - RPC는 서비스 간 내부 트래픽에, REST는 공개·브라우저 대상 API에 강점을 갖는 경향.

> [!warning] 주의점
> - "로컬 호출처럼 보인다"는 추상화는 실제 네트워크 실패 모드(타임아웃, 부분 실패, 재시도)를 가린다.
> - 클라이언트와 서버 사이의 강한 스키마 결합은 규율 있는 버저닝을 요구한다.

현대 RPC 스택에는 [[gRPC]], Thrift, JSON-RPC 등이 있다. [[Microservices]] 아키텍처에서 광범위하게 쓰인다.

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
