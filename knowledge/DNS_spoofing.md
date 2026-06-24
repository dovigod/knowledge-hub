---
id: 019ea149-98cd-7572-a3d8-bb919b8f3c3e
name: DNS spoofing
aliases:
  - DNS hijacking
  - DNS poisoning
  - DNS 스푸핑
  - DNS 하이재킹
updated_at: '2026-06-07T08:53:35.566Z'
summary: >-
  Application-layer attack that returns forged DNS answers, sending victims to
  attacker-controlled IPs despite typing the correct hostname.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# DNS spoofing

## Overview

DNS spoofing returns false `name → IP` answers, so a victim who types the correct hostname (e.g. `bank.local`) is silently sent to an attacker's IP. It sits at L7 of the [[OSI 7-layer model]] and is often the second stage chained after a routing-level hijack like [[BGP hijacking]] — first capture the path to the real DNS server, then answer queries with lies.

> [!note] In the family
> [[BGP hijacking]] hijacks the route. DNS spoofing hijacks the name. DNAT ([[iptables NAT]]) hijacks the rewrite. [[ARP spoofing]] hijacks the local link. Same idea, different layer.

## Notes

### Mechanics
- **Cache poisoning:** inject forged records into a recursive resolver's cache.
- **Path interception:** sit in front of the resolver (via [[ARP spoofing]] on LAN, or [[BGP hijacking]] for upstream DNS).
- **Rogue local resolver:** Evil Twin AP hands out a malicious DNS server via DHCP.

### Chained with BGP — the [[MyEtherWallet 2018 BGP hijacking]] pattern
1. BGP-hijack the prefix that hosts the authoritative DNS server.
2. Run a spoofed DNS server that returns attacker IPs for the target domain.
3. Victims who type `myetherwallet.com` get the attacker's IP.
4. Phishing page captures private keys / credentials.

### Defenses
- **DNSSEC**: signed DNS responses; resolvers verify signatures.
- **DoH / DoT** (DNS over HTTPS / TLS): encrypts the DNS query path itself.
- **[[TLS]] certificate validation at the endpoint**: the killer defense — even if DNS is fully compromised, the phishing IP cannot present a CA-signed cert for the real hostname, so the browser refuses (unless the user clicks through the warning).
- **DHCP snooping** + dynamic ARP inspection on LAN to prevent rogue DHCP/DNS injection.

> [!tip] Why TLS works regardless
> DNS answers "where is this name?" TLS answers "is this peer really that name?" The second check uses CA-signed identity, which an off-path attacker cannot forge — see [[MyEtherWallet 2018 BGP hijacking]] for a live example.

---

## 한국어

### 개요

DNS 스푸핑은 거짓 `이름 → IP` 응답을 돌려준다. 피해자가 정확한 호스트명(예: `bank.local`)을 입력해도 공격자 IP로 조용히 보내진다. [[OSI 7-layer model]] L7에 위치하며, [[BGP hijacking]] 같은 라우팅 계층 하이재킹과 자주 연쇄된다 — 먼저 진짜 DNS 서버로 가는 경로를 가로채고, 그다음 질의에 거짓으로 답한다.

> [!note] 가족 관계
> [[BGP hijacking]]은 경로를, DNS 스푸핑은 이름을, DNAT ([[iptables NAT]])는 재작성을, [[ARP spoofing]]은 로컬 링크를 장악. 같은 아이디어, 다른 계층.

### 노트

#### 작동 방식
- **캐시 포이즈닝:** 재귀 리졸버의 캐시에 위조 레코드 주입.
- **경로 가로채기:** 리졸버 앞에 자리 잡기 (LAN에선 [[ARP spoofing]]으로, 상위 DNS는 [[BGP hijacking]]으로).
- **악성 로컬 리졸버:** Evil Twin AP가 DHCP로 악성 DNS 서버를 나눠줌.

#### BGP와 연쇄 — [[MyEtherWallet 2018 BGP hijacking]] 패턴
1. 권위 DNS 서버가 있는 대역을 BGP 하이재킹.
2. 위조 DNS 서버를 돌려 대상 도메인에 공격자 IP를 응답.
3. `myetherwallet.com`을 친 피해자가 공격자 IP에 도달.
4. 피싱 페이지가 개인키/자격증명을 탈취.

#### 방어
- **DNSSEC**: 서명된 DNS 응답, 리졸버가 서명 검증.
- **DoH / DoT** (DNS over HTTPS / TLS): DNS 질의 경로 자체 암호화.
- **엔드포인트의 [[TLS]] 인증서 검증**: 결정타 — DNS가 완전히 뚫려도 피싱 IP는 진짜 호스트명의 CA 서명 인증서를 제시할 수 없으므로 브라우저가 거부 (사용자가 경고를 클릭해 넘기지 않는 한).
- LAN에선 DHCP snooping + 동적 ARP 검사로 악성 DHCP/DNS 주입 차단.

> [!tip] TLS가 계층 무관하게 작동하는 이유
> DNS는 "이 이름은 어디인가"를 답하고, TLS는 "이 상대가 정말 그 이름인가"를 답한다. 두 번째 검증은 CA 서명 신원을 쓰는데 off-path 공격자는 위조 불가 — 실제 사례는 [[MyEtherWallet 2018 BGP hijacking]] 참고.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
