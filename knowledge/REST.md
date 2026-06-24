---
id: 019e8db9-8975-70cb-ab9a-c3b5b0feebaf
name: REST
aliases:
  - REST API
  - RESTful
  - rest
updated_at: '2026-06-03T13:43:27.349Z'
summary: >-
  An architectural style for networked APIs that maps resources to URLs and
  operations to HTTP verbs, typically using JSON over HTTP/1.1.
sources:
  - 019e8db3-b83e-766f-9304-a0ab2827ffaa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# REST

## Overview

> [!note] TL;DR
> REST (Representational State Transfer) is an architectural style for HTTP APIs where resources have URLs and operations map to HTTP verbs (`GET`, `POST`, `PUT`, `DELETE`). Most REST APIs today use [[JSON]] over HTTP/1.1, making them universally debuggable but less efficient than [[gRPC]].

## Notes

> [!tip] Strengths
> - **Universal tooling** — `curl`, browsers, Postman, every language's standard HTTP library works out of the box.
> - **Human-readable wire format** — debug with Wireshark or browser devtools directly.
> - **Ecosystem and convention** — OpenAPI/Swagger, vast documentation culture.
> - **Best for public APIs** intended for external developers.

> [!warning] Weaknesses vs [[gRPC]]
> - No enforced schema — field renames/typos surface at runtime, not compile time.
> - JSON parsing is CPU-expensive on high-traffic servers.
> - HTTP/1.1 means one request per connection; browsers compensate with 6+ parallel connections.
> - Streaming requires bolting on WebSocket or Server-Sent Events.
> - Status semantics are coarse (4xx/5xx) compared to gRPC's typed codes.

> [!tip] When to choose REST
> Browser-facing APIs, public APIs, anywhere `curl`-level debuggability matters more than raw throughput.

---

## 한국어

### 개요

> [!note] 요약
> REST(Representational State Transfer)는 리소스를 URL에, 작업을 HTTP 동사(`GET`, `POST`, `PUT`, `DELETE`)에 매핑하는 HTTP API 아키텍처 스타일입니다. 오늘날 대부분의 REST API는 HTTP/1.1 위에서 [[JSON]]을 사용하며, 보편적으로 디버깅 가능하지만 [[gRPC]]보다는 비효율적입니다.

### 노트

> [!tip] 강점
> - **보편적 도구 지원** — `curl`, 브라우저, Postman, 모든 언어의 표준 HTTP 라이브러리가 즉시 동작.
> - **사람이 읽을 수 있는 와이어 포맷** — Wireshark나 브라우저 devtools로 직접 디버깅.
> - **생태계와 관습** — OpenAPI/Swagger, 방대한 문서화 문화.
> - **공개 API에 최적** — 외부 개발자 대상.

> [!warning] [[gRPC]] 대비 약점
> - 강제 스키마 없음 — 필드 이름 변경/오타가 컴파일 타임이 아닌 런타임에 발견됨.
> - 고트래픽 서버에서 JSON 파싱이 CPU를 많이 먹음.
> - HTTP/1.1은 연결당 요청 하나; 브라우저가 6개 이상 병렬 연결로 보완.
> - 스트리밍은 WebSocket이나 Server-Sent Events를 별도로 붙여야 함.
> - 상태 코드 의미가 거칠음(4xx/5xx) — gRPC의 타입 코드와 비교됨.

> [!tip] REST를 선택할 때
> 브라우저 대상 API, 공개 API, 순수 처리량보다 `curl` 수준의 디버깅 편의가 중요한 곳.

## Sources

- [[raw/conversations/019e8db3-b83e-766f-9304-a0ab2827ffaa|019e8db3-b83e-766f-9304-a0ab2827ffaa]]
