---
id: 019ea149-87a7-77ac-89ca-525c736faca6
name: BGP hijacking
aliases:
  - BGP hijack
  - BGP 하이재킹
  - prefix hijacking
updated_at: '2026-06-07T08:53:31.176Z'
summary: >-
  Routing-layer attack where a malicious AS advertises false BGP routes to
  attract and redirect internet traffic to itself.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# BGP hijacking

## Overview

BGP hijacking is a Layer-3 attack ([[OSI 7-layer model]]) where an attacker abuses the Border Gateway Protocol — the protocol routers use to share "how to reach IP prefix X" — by advertising routes they don't own. Other routers, trusting the announcement, send traffic to the attacker instead of the legitimate destination.

> [!note] One-line
> Whoever controls the route controls the traffic. BGP hijacking is the route-control version of that idea, alongside [[ARP spoofing]] (L2), DNAT hijacking ([[iptables NAT]]), and [[DNS spoofing]] (L7).

## Notes

### How it works
1. Attacker announces a **more-specific** prefix (longer prefix length wins), e.g. announcing `203.0.113.0/24` when the legit owner only announces `203.0.113.0/22`.
2. Other ASes accept the more-specific route as the best path.
3. Traffic destined for that prefix now flows to the attacker.
4. Attacker can drop, log, modify, or chain into a [[DNS spoofing]] / phishing step.

### Famous example — MyEtherWallet 2018
See [[MyEtherWallet 2018 BGP hijacking]]: attackers BGP-hijacked Amazon Route53 DNS prefixes, then ran spoofed DNS to send victims to a phishing site, stealing ~215 ETH (~$150,000). The only thing that stopped most victims was the **TLS certificate warning** ([[TLS]]) — those who clicked "proceed anyway" lost funds.

### Defenses
- **RPKI** (Resource Public Key Infrastructure): cryptographic proof of who is allowed to announce which prefix. Routers reject unauthorized announcements.
- **BGPsec** (still rare in practice).
- **Monitoring** services (e.g. BGP route monitors, Cloudflare/Qrator alerts) to detect anomalies.
- **[[TLS]] / [[mTLS]]** at the endpoint: even if the route is hijacked, the attacker cannot forge a CA-signed certificate for the real hostname, so clients refuse to connect — *as long as users don't bypass certificate warnings*.

> [!warning] Why TLS is the last line
> Lower-layer attacks (BGP, DNS, ARP) all answer "where do I send traffic?" TLS answers "is the peer who they claim to be?" The first three can all be wrong simultaneously and TLS still blocks the attack — unless the user ignores the warning.

---

## 한국어

### 개요

BGP 하이재킹은 [[OSI 7-layer model]] L3 공격으로, 라우터들이 "IP 대역 X��� 도달하는 방법"을 공유할 때 쓰는 BGP를 악용한다. 공격자가 자기 것이 아닌 경로를 광고하면, 그 광고를 신뢰한 다른 라우터들이 정상 목적지가 아닌 공격자에게 트래픽을 보낸다.

> [!note] 한 줄
> 경로를 쥔 자가 트래픽을 쥔다. BGP 하이재킹은 이 아이디어의 "라우팅 장악" 판본 — [[ARP spoofing]] (L2), DNAT 하이재킹 ([[iptables NAT]]), [[DNS spoofing]] (L7)과 한 가족.

### 노트

#### 작동 방식
1. 공격자가 **더 좁은** 대역을 광고한다(더 긴 prefix가 이김). 예: 정상 소유자가 `203.0.113.0/22`만 광고할 때 공격자가 `203.0.113.0/24`를 광고.
2. 다른 AS들이 더 좁은 경로를 최적 경로로 채택.
3. 그 대역으로 가는 트래픽이 공격자에게 흘러감.
4. 공격자가 폐기/기록/변조하거나 [[DNS spoofing]]/피싱 단계로 연쇄.

#### 유명 사례 — MyEtherWallet 2018
[[MyEtherWallet 2018 BGP hijacking]] 참고: 공격자가 Amazon Route53 DNS 대역을 BGP 하이재킹하고, 위조 DNS로 피해자를 피싱 사이트에 보내 약 215 ETH(~$150,000)를 탈취. 대다수가 막힌 유일한 이유는 **TLS 인증서 경고** ([[TLS]]) — "그래도 진행"을 누른 사람만 털림.

#### 방어
- **RPKI** (Resource Public Key Infrastructure): 어느 AS가 어느 대역을 광고할 자격이 있는지 암호학적 증명. 무권한 광고를 라우터가 거부.
- **BGPsec** (실제론 아직 드묾).
- BGP 라우트 모니터링(Cloudflare/Qrator 같은 서비스)으로 이상 탐지.
- 엔드포인트에서 **[[TLS]] / [[mTLS]]**: 경로가 하이재킹돼도 공격자는 진짜 호스트명의 CA 서명 인증서를 위조 못 함 → 클라이언트가 연결 거부. *단, 사용자가 인증서 경고를 무시하지 않는 한.*

> [!warning] TLS가 마지막 방어선인 이유
> 하위 계층 공격(BGP, DNS, ARP)은 모두 "트래픽을 어디로 보내는가"를 답한다. TLS는 "상대가 진짜 그 사람인가"를 답한다. 앞의 셋이 동시에 틀려도 TLS는 막을 수 있다 — 사용자가 경고를 무시하지 않는 한.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
