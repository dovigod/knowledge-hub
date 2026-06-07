---
id: 019ea11d-90bf-70bb-9018-fa68c0741e5b
name: OSI Model
aliases:
  - OSI
  - OSI 7 Layer
  - OSI Reference Model
updated_at: '2026-06-07T08:05:29.919Z'
summary: >-
  Seven-layer conceptual model that describes how network protocols communicate,
  from physical signaling (L1) to application data (L7).
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OSI Model

## Overview

The OSI (Open Systems Interconnection) Model is a conceptual framework standardized by ISO that divides network communication into seven distinct layers, each with specific responsibilities. While real-world stacks (like TCP/IP) don't map cleanly to all seven, the model remains the lingua franca for discussing where a protocol or security concern operates.

> [!example] The seven layers
> L1 Physical → L2 Data Link → L3 Network → L4 Transport → L5 Session → L6 Presentation → L7 Application

## Notes

- **L1 Physical**: cables, radio, raw bits
- **L2 Data Link**: Ethernet, MAC addresses, switches
- **L3 Network**: [[IP]], routing, ICMP, firewalls operate here
- **L4 Transport**: [[TCP]], [[UDP]], port numbers — stateful filtering happens here
- **L5 Session**: session establishment (rarely a distinct layer in practice)
- **L6 Presentation**: encryption, serialization (often folded into L7)
- **L7 Application**: HTTP, [[SSH]], DNS, gRPC — content-aware policies
- **Security implications**:
  - L3/L4 filtering = traditional firewalls
  - L7 filtering = WAF, API gateway, [[Cilium]] policies
- Related: [[TCP/IP Model]], [[Cilium]], [[Firewall]]

> [!tip] Mnemonic
> "Please Do Not Throw Sausage Pizza Away" — Physical, Data Link, Network, Transport, Session, Presentation, Application.

---

## 한국어

### 개요

OSI(Open Systems Interconnection) 모델은 ISO에서 표준화한 개념적 프레임워크로, 네트워크 통신을 7개의 명확한 계층으로 나누며 각 계층은 특정 책임을 갖는다. 실제 스택(TCP/IP 같은)이 7개 계층에 완벽히 매핑되지는 않지만, 프로토콜이나 보안 관심사가 어디에서 동작하는지 논의할 때의 공용어로 남아 있다.

> [!example] 7개 계층
> L1 물리 → L2 데이터 링크 → L3 네트워크 → L4 전송 → L5 세션 → L6 표현 → L7 응용

### 노트

- **L1 물리**: 케이블, 무선, 원시 비트
- **L2 데이터 링크**: Ethernet, MAC 주소, 스위치
- **L3 네트워크**: [[IP]], 라우팅, ICMP, 방화벽 동작
- **L4 전송**: [[TCP]], [[UDP]], 포트 번호 — 상태 기반 필터링
- **L5 세션**: 세션 수립 (실제로는 별개 계층이 드묾)
- **L6 표현**: 암호화, 직렬화 (보통 L7에 포함됨)
- **L7 응용**: HTTP, [[SSH]], DNS, gRPC — 컨텐츠 인식 정책
- **보안 시사점**:
  - L3/L4 필터링 = 전통 방화벽
  - L7 필터링 = WAF, API 게이트웨이, [[Cilium]] 정책
- 관련: [[TCP/IP Model]], [[Cilium]], [[Firewall]]

> [!tip] 암기법
> "Please Do Not Throw Sausage Pizza Away" — Physical, Data Link, Network, Transport, Session, Presentation, Application.

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
