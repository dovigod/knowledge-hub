---
id: 019ea149-ad4c-7175-86f5-18116c076eb2
name: ARP spoofing
aliases:
  - ARP poisoning
  - ARP 스푸핑
  - arp spoof
updated_at: '2026-06-07T08:53:40.812Z'
summary: >-
  Layer-2 attack where the attacker forges ARP replies on a LAN, causing victims
  to send traffic to the attacker instead of the real MAC.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# ARP spoofing

## Overview

ARP spoofing is a Layer-2 attack ([[OSI 7-layer model]]) on a local network. ARP (Address Resolution Protocol) maps IPs to MAC addresses on the same LAN. An attacker broadcasts forged "IP X is at MY MAC" replies; victims update their ARP caches and start sending traffic for X (often the gateway) to the attacker.

> [!note] In the family
> Whoever controls the route controls the traffic. ARP spoofing is the LAN/L2 version of [[BGP hijacking]] (L3 / internet-scale), DNAT hijacking ([[iptables NAT]]), and [[DNS spoofing]] (L7).

## Notes

### Typical scenario
1. Attacker on the same LAN sends gratuitous ARP: "the gateway 192.168.0.1 is at AA:BB:CC:DD:EE:FF" (attacker's MAC).
2. Victim's ARP table updates.
3. All outbound traffic from the victim flows through the attacker, who can MITM, log, or modify it.
4. Often chained with [[DNS spoofing]] or HTTPS-strip attempts.

### Defenses
- **Dynamic ARP Inspection (DAI)** on managed switches: validate ARP replies against DHCP snooping bindings.
- **DHCP snooping**: trust only DHCP replies on designated ports.
- **Port Security**: limit MAC addresses per port, disable unknown MACs.
- **802.1X**: authenticate devices before they're allowed on the LAN.
- **[[TLS]] / [[mTLS]]** end-to-end: even if MITM succeeds at L2, the attacker can't forge a valid certificate for the real server.

> [!warning] Same conceptual move as DNAT hijacking
> ARP spoofing is to a LAN what a malicious [[iptables NAT|DNAT]] rule is to a Docker host: it convinces packets to follow a wrong path. The defense families overlap heavily — minimize who can manipulate routing primitives, and use [[TLS]] to make the redirection useless.

---

## 한국어

### 개요

ARP 스푸핑은 로컬 네트워크에서의 [[OSI 7-layer model]] L2 공격이다. ARP(Address Resolution Protocol)는 같은 LAN에서 IP를 MAC 주소로 해석한다. 공격자가 "IP X는 내 MAC이다"라는 위조 ARP 응답을 뿌리면, 피해자가 ARP 캐시를 갱신하고 X(주로 게이트웨이)로 가는 트래픽을 공격자에게 보내기 시작.

> [!note] 가족 관계
> 경로를 쥔 자가 트래픽을 쥔다. ARP 스푸핑은 [[BGP hijacking]](L3 / 인터넷 스케일), DNAT 하이재킹 ([[iptables NAT]]), [[DNS spoofing]] (L7)의 LAN/L2 판본.

### 노트

#### 전형적 시나리오
1. 같은 LAN의 공격자가 gratuitous ARP 전송: "게이트웨이 192.168.0.1은 AA:BB:CC:DD:EE:FF"(공격자 MAC).
2. 피해자 ARP 테이블 갱신.
3. 피해자의 나가는 트래픽이 모두 공격자를 거쳐감 — MITM/기록/변조 가능.
4. [[DNS spoofing]]이나 HTTPS-strip 시도와 자주 연쇄.

#### 방어
- 관리형 스위치의 **Dynamic ARP Inspection (DAI)**: DHCP snooping 바인딩으로 ARP 응답 검증.
- **DHCP snooping**: 지정된 포트에서만 DHCP 응답 신뢰.
- **Port Security**: 포트별 MAC 제한, 알 수 없는 MAC 차단.
- **802.1X**: LAN 합류 전 장치 인증.
- **[[TLS]] / [[mTLS]]** 종단간: L2에서 MITM이 성공해도 공격자는 진짜 서버의 유효 인증서를 위조 못 함.

> [!warning] DNAT 하이재킹과 같은 개념적 수법
> ARP 스푸핑이 LAN에 대해 하는 일은 악성 [[iptables NAT|DNAT]] 규칙이 Docker 호스트에 대해 하는 일과 같다 — 패킷을 잘못된 경로로 유도. 방어 가족도 크게 겹친다: 라우팅 원시 동작을 만질 수 있는 주체를 최소화하고, [[TLS]]로 우회 자체를 무의미하게.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
