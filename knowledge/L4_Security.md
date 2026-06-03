---
id: 019e8d54-00dd-76e1-94f4-46c917f26d76
name: L4 Security
aliases:
  - Layer 4 Security
  - Transport Layer Security (networking)
updated_at: '2026-06-03T11:52:33.245Z'
summary: >-
  Transport layer (TCP/UDP) network security focused on ports, protocols, and
  connection-level filtering.
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# L4 Security

## Overview
L4 (Layer 4) security operates at the transport layer of the OSI model, controlling traffic based on TCP/UDP ports, protocols, and connection state. It is the traditional domain of firewalls and basic network policies.

## Notes
- Filters based on IP addresses, ports, and protocols (TCP/UDP)
- Implemented via stateful firewalls, iptables, security groups
- Cannot inspect application payload content
- Examples: "Allow TCP port 443 from 10.0.0.0/8"
- Faster than L7 but less granular
- Kubernetes NetworkPolicy operates at L3/L4

---

## 한국어

### 개요
L4 (Layer 4) 보안은 OSI 모델의 전송 계층에서 동작하며, TCP/UDP 포트, 프로토콜, 연결 상태를 기반으로 트래픽을 제어합니다. 방화벽과 기본 네트워크 정책의 전통적인 영역입니다.

### 노트
- IP 주소, 포트, 프로토콜(TCP/UDP) 기반 필터링
- 스테이트풀 방화벽, iptables, 시큐리티 그룹으로 구현
- 애플리케이션 페이로드 내용은 검사 불가
- 예: "10.0.0.0/8에서 TCP 포트 443 허용"
- L7보다 빠르지만 세분화 정도가 낮음
- Kubernetes NetworkPolicy는 L3/L4에서 동작

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
