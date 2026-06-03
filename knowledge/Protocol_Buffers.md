---
id: 019e8db0-d6ed-73c0-abd5-ccda5c5c2da9
name: Protocol Buffers
aliases:
  - Protobuf
  - Protocol Buffers
  - proto
  - proto3
  - protobuf
  - protocol-buffers
updated_at: '2026-06-03T13:42:44.427Z'
summary: >-
  Language-neutral, binary interface definition language and serialization
  format developed by Google, commonly used as the payload format for gRPC.
sources:
  - 019e8daf-7ac6-70aa-8dac-2f6377d5435b
  - 019e8db3-b83e-766f-9304-a0ab2827ffaa
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

### Why the wire format is small

Each field is identified by a **numeric tag** (declared as `= 1`, `= 2`, … in the `.proto`) rather than a string name. Integers use **varint** encoding, so small numbers take 1–2 bytes instead of the 4–8 of a fixed-width int (or the variable-length text of [[JSON]]).

> [!example] Same payload, two encodings
> ```
> JSON:     {"id":42,"name":"justin","email":"j@x.io"}     → 43 bytes
> Protobuf: 0x08 0x2A 0x12 0x06 justin 0x1A 0x06 j@x.io    → ~20 bytes
> ```

### Why parsing is cheap

[[JSON]] parsing tokenizes text, branches on `{` / `"` / digits, and infers types at runtime. Protobuf already knows from the generated code that "tag 2 is a `string` called `name`", so decoding is mostly bounds-checked memory copies — a meaningful CPU win on high-traffic servers where (de)serialization dominates.

### Schema evolution rules

> [!warning] Tag numbers are the contract
> Adding new fields with new tag numbers is safe (old readers ignore them). **Reusing or renumbering existing tags is forbidden** — it silently corrupts data for clients built against the old schema.

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

#### 와이어 포맷이 작은 이유

각 필드는 문자열 이름이 아니라 **숫자 태그**(`.proto`에서 `= 1`, `= 2`, …로 선언)로 식별된다. 정수는 **varint** 인코딩을 쓰므로 작은 값은 고정폭 4~8바이트(또는 [[JSON]]의 가변 길이 텍스트) 대신 1~2바이트면 충분하다.

> [!example] 같은 데이터, 두 가지 인코딩
> ```
> JSON:     {"id":42,"name":"justin","email":"j@x.io"}     → 43 bytes
> Protobuf: 0x08 0x2A 0x12 0x06 justin 0x1A 0x06 j@x.io    → ~20 bytes
> ```

#### 파싱이 싼 이유

[[JSON]] 파싱은 텍스트를 토크나이징하고 `{` / `"` / 숫자에 따라 분기하며 런타임에 타입을 추론한다. 반면 protobuf는 생성된 코드가 "태그 2번 = `name`, `string`" 임을 이미 알고 있어, 디코딩이 대부분 경계 검사 + 메모리 복사로 끝난다. 고트래픽 서버에서 (역)직렬화가 CPU를 많이 먹기 때문에 실질적인 차이로 나타난다.

#### 스키마 진화 규칙

> [!warning] 태그 번호가 곧 계약이다
> 새 태그 번호로 필드를 추가하는 것은 안전하다(구버전 리더는 모르는 필드를 무시한다). **기존 태그를 재사용하거나 번호를 바꾸는 것은 금지** — 구 스키마로 빌드된 클라이언트의 데이터가 조용히 깨진다.

[[Microservices]] 통신을 위해 [[gRPC]]와 자주 묶여 쓰이지만, 일반적으로 [[JSON]]을 쓰는 자리에 단독 직렬화 포맷으로도 사용 가능하다.

## Sources

- [[raw/conversations/019e8daf-7ac6-70aa-8dac-2f6377d5435b|019e8daf-7ac6-70aa-8dac-2f6377d5435b]]
- [[raw/conversations/019e8db3-b83e-766f-9304-a0ab2827ffaa|019e8db3-b83e-766f-9304-a0ab2827ffaa]]
