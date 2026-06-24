---
id: 019ea14b-73cd-7567-b7f9-7f688e0a717f
name: MyEtherWallet 2018 BGP hijacking
aliases:
  - MEW 2018 hijack
  - MEW 2018 사건
  - MEW BGP
  - MyEtherWallet hijack
updated_at: '2026-06-07T08:55:37.165Z'
summary: >-
  April 2018 attack chaining BGP hijacking of AWS DNS, DNS spoofing, and a
  phishing site; about 215 ETH (~$150K) stolen — TLS warnings were the only
  thing that saved most users.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# MyEtherWallet 2018 BGP hijacking

## Overview

On April 24, 2018, attackers chained a [[BGP hijacking]] of Amazon Route53 prefixes, a [[DNS spoofing]] step, and a phishing site to steal cryptocurrency from MyEtherWallet (MEW) users. About 215 ETH (~$150,000 at the time) was drained. The decisive detail: the phishing site had no valid TLS certificate, so every browser threw a warning — the only users who lost funds were those who clicked **"proceed anyway."** It's a textbook illustration of why [[TLS]] is the last line of defense.

> [!note] One-sentence lesson
> The attackers fully controlled the route and the name resolution; **certificate validation was the single thing standing between the user and a drained wallet.**

## Notes

### Three-stage attack chain
1. **[[BGP hijacking]] (L3 of [[OSI 7-layer model]]):** attackers announced more-specific routes for AWS Route53 DNS prefixes. Other ASes accepted the more-specific path → DNS traffic for Route53 flowed to attackers.
2. **[[DNS spoofing]] (L7):** the hijacked DNS responded `myetherwallet.com → Russian attacker IP`.
3. **Phishing site:** victims who typed the correct URL landed on a clone that captured private keys.

### The TLS save
The attacker's server didn't have a CA-signed certificate for `myetherwallet.com` (impossible to obtain — they don't have the domain owner's CA-validated control). Browsers showed the "connection not private" warning. Almost all users bounced off. The ~215 ETH lost came from users who clicked through the warning.

### Defense lessons cited from the discussion
- **Axis B (make takeover useless) is the killer:** [[TLS]] / [[mTLS]] turned a complete route + name compromise into a near-zero financial loss for most users.
- **Axis A (prevent takeover):** RPKI to prevent BGP hijacks at source; DNSSEC for DNS integrity.
- **Axis C (detect changes):** monitor BGP announcements, anomaly detection on route changes.
- **UX:** make TLS warnings hard to bypass (HSTS, certificate pinning) so the "proceed anyway" footgun is removed.

> [!warning] The general pattern
> Whoever controls the route controls the traffic — but only as far as identity verification lets them. [[BGP hijacking]] + [[DNS spoofing]] + [[ARP spoofing]] + DNAT ([[iptables NAT]]) all answer "where do I send the bytes?" [[TLS]] answers "is the receiver who they claim?" — and that question is what the MEW victims either trusted (saved) or overrode (lost).

---

## 한국어

### 개요

2018년 4월 24일, 공격자들이 Amazon Route53 대역의 [[BGP hijacking]], [[DNS spoofing]], 피싱 사이트를 연쇄해 MyEtherWallet(MEW) 사용자의 암호화폐를 탈취했다. 약 215 ETH(당시 ~$150,000) 손실. 결정적 디테일: 피싱 사이트엔 유효한 TLS 인증서가 없어서 브라우저가 모두 경고를 띄웠고 — **"그래도 진행"을 누른 사용자만 자금을 잃었다.** [[TLS]]가 왜 마지막 방어선인지 보여주는 교과서적 사례.

> [!note] 한 줄 교훈
> 공격자가 경로와 이름 해석을 완전히 장악했지만, **인증서 검증이 사용자와 빈 지갑 사이에 남은 유일한 방패였다.**

### 노트

#### 3단계 공격 체인
1. **[[BGP hijacking]] ([[OSI 7-layer model]] L3):** 공격자가 AWS Route53 DNS 대역에 대해 더 좁은(more-specific) 경로를 광고. 다른 AS들이 그 경로를 채택 → Route53로 갈 DNS 트래픽이 공격자에게.
2. **[[DNS spoofing]] (L7):** 가로챈 DNS가 `myetherwallet.com → 러시아 공격자 IP`로 응답.
3. **피싱 사이트:** 정확한 URL을 친 피해자가 클론 사이트에 도달, 개인키가 탈취됨.

#### TLS의 구원
공격자 서버는 `myetherwallet.com`에 대한 CA 서명 인증서를 갖지 못함(불가능 — 도메인 소유자의 CA 검증된 통제권이 없음). 브라우저가 "이 연결은 비공개가 아닙니다" 경고. 거의 모든 사용자가 튕겨 나갔다. 잃은 ~215 ETH는 경고를 클릭해 넘긴 사용자들에서 나옴.

#### 토론에서 도출된 방어 교훈
- **축 B(장악당해도 무용지물)가 결정타:** [[TLS]] / [[mTLS]]가 완전한 경로+이름 장악을 대다수 사용자에겐 거의 0의 금전 손실로 바꿈.
- **축 A(장악 자체 막기):** BGP 하이재킹은 RPKI로 원천 차단; DNS는 DNSSEC로 무결성 확보.
- **축 C(변경 탐지):** BGP 광고 모니터링, 경로 변경 이상 탐지.
- **UX:** TLS 경고를 무시하기 어렵게(HSTS, 인증서 피닝) — "그래도 진행" 함정을 제거.

> [!warning] 일반 패턴
> 경로를 쥔 자가 트래픽을 쥔다 — 단, 신원 검증이 허락하는 만큼만. [[BGP hijacking]] + [[DNS spoofing]] + [[ARP spoofing]] + DNAT ([[iptables NAT]])은 모두 "바이트를 어디로 보내는가"를 답한다. [[TLS]]는 "받는 자가 자기가 주장하는 자인가"를 답한다 — 이 질문이 MEW 피해자들이 신뢰했거나(살았거나) 무시한(잃은) 바로 그것.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
