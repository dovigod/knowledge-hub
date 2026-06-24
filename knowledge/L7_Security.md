---
id: 019e8d54-0f2f-711e-bbd2-3baffaad0b58
name: L7 Security
aliases:
  - Application Layer Security
  - Layer 7 Security
updated_at: '2026-06-03T11:52:36.911Z'
summary: >-
  Application layer security that inspects and filters traffic based on
  protocol-specific content like HTTP methods, paths, and headers.
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# L7 Security

## Overview
L7 (Layer 7) security operates at the application layer of the OSI model, providing deep inspection and policy enforcement based on application-specific protocols such as HTTP, gRPC, Kafka, and DNS.

## Notes
- Inspects application payload content (HTTP methods, paths, headers)
- Examples: "Allow GET /api/v1/users but block DELETE"
- Implemented by WAF, API gateways, service meshes (Istio, Linkerd), Cilium
- More CPU-intensive than L4 filtering
- Enables fine-grained API-level authorization
- Protocol-aware: HTTP, gRPC, Kafka, DNS, etc.

---

## 한국어

### 개요
L7 (Layer 7) 보안은 OSI 모델의 애플리케이션 계층에서 동작하며, HTTP, gRPC, Kafka, DNS 같은 애플리케이션별 프로토콜을 기반으로 심층 검사와 정책 적용을 제공합니다.

### 노트
- 애플리케이션 페이로드 내용(HTTP 메서드, 경로, 헤더) 검사
- 예: "GET /api/v1/users 허용, DELETE 차단"
- WAF, API 게이트웨이, 서비스 메시(Istio, Linkerd), Cilium으로 구현
- L4 필터링보다 CPU 사용량이 많음
- 세분화된 API 수준 권한 부여 가능
- 프로토콜 인식: HTTP, gRPC, Kafka, DNS 등

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
