---
id: 019e8db0-d6ed-73c0-abd5-ccda5c5c2da9
name: Protocol Buffers
aliases:
  - Protobuf
  - proto
  - protobuf
  - protocol-buffers
updated_at: '2026-06-03T13:33:57.357Z'
summary: >-
  Language-neutral, binary interface definition language and serialization
  format developed by Google, commonly used as the payload format for gRPC.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Protocol Buffers

## Overview

Protocol Buffers (protobuf) is Google's language-neutral, platform-neutral interface definition language and binary serialization format. It defines message schemas in `.proto` files, then generates strongly-typed client and server code for many languages — making it the default payload and contract layer for [[gRPC]].

> [!note] Role
> Protobuf is both an IDL (schema definition) and a wire format (compact binary encoding). The schema is the source of truth; generated code enforces it.

## Notes

> [!tip] Strengths
> - Compact binary wire format → smaller payloads than [[JSON]]
> - Strongly typed contracts, enabling compile-time safety
> - Forward/backward compatibility through field numbering
> - Polyglot code generation across most major languages

> [!warning] Trade-offs
> - Not human-readable on the wire (harder ad-hoc debugging vs [[JSON]])
> - Requires a code-gen toolchain in the build pipeline
> - Schema evolution rules must be followed strictly to stay compatible

Commonly paired with [[gRPC]] for [[Microservices]] communication, but usable standalone as a serialization format wherever [[JSON]] would otherwise be used.

---

## 한국어

### 개요

Protocol Buffers(protobuf)는 구글이 만든 언어 중립적, 플랫폼 중립적인 인터페이스 정의 언어이자 바이너리 직렬화 포맷이다. `.proto` 파일에 메시지 스키마를 정의하면 여러 언어로 강타입 클라이언트/서버 코드를 자동 생성해 주며, [[gRPC]]의 기본 페이로드 및 계약 계층으로 쓰인다.

> [!note] 역할
> protobuf는 IDL(스키마 정의)이자 와이어 포맷(컴팩트한 바이너리 인코딩)이다. 스키마가 단일 진실 공급원이고, 생성된 코드가 이를 강제한다.

### 노트

> [!tip] 장점
> - 컴팩트한 바이너리 포맷 → [[JSON]]보다 작은 페이로드
> - 강타입 계약으로 컴파일 타임 안전성 확보
> - 필드 번호 기반 전/후방 호환성
> - 주요 언어 대부분에 대한 폴리글랏 코드 생성

> [!warning] 단점
> - 와이어에서 사람이 읽을 수 없음 ([[JSON]] 대비 즉석 디버깅 어려움)
> - 빌드 파이프라인에 코드 생성 툴체인 필요
> - 호환성 유지하려면 스키마 진화 규칙을 엄격히 따라야 함

[[Microservices]] 통신을 위해 [[gRPC]]와 자주 묶여 쓰이지만, 일반적으로 [[JSON]]을 쓰는 자리에 단독 직렬화 포맷으로도 사용 가능하다.

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
