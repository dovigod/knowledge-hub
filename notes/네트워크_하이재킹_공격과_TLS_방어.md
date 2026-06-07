---
id: 019ea125-2c28-7219-9a8e-a4ce1f381475
title: 네트워크 하이재킹 공격과 TLS 방어
topics:
  - security
  - hijacking
  - bgp
  - dns
  - dnat
  - mitm
  - tls
  - mtls
  - 인증서 검증
  - mew-2018
  - infra
sources:
  - 019ea121-0888-7779-b38c-63c0aa2fc563
created_at: '2026-06-07T08:13:48.455Z'
updated_at: '2026-06-07T08:13:48.455Z'
---
## NAT/DNAT Hijacking Demo

Insight: whoever controls the NAT table controls the packet's destination (root/privileged operation). Demo: command `iptables -t nat -I DOCKER 1 ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination <B_IP>:80`. Finding: on Mac, localhost:8080 still goes to A ([[OrbStack]] docker-proxy bypasses [[iptables]]). Path 2 VM_IP:8080→B (hijacking succeeds, traverses PREROUTING→DOCKER DNAT). That is, [[DNAT]] manipulation only intercepts traffic when that path actually passes through iptables. Restore precisely with `-D`.

## Network Path Hijacking — Attacks & Defenses

Essence: whoever seizes the path (routing / [[NAT]] / address resolution) seizes the traffic. Attacks: malicious privileged container sidecar [[MITM]], [[Docker]] socket exposure, cloud metadata hijacking, [[ARP]]/[[DHCP]] spoofing, [[DNS]]/[[BGP]] hijacking, [[Evil Twin]]. Defenses: Axis A least privilege (no `--privileged`, `--cap-drop=ALL`, K8s [[NetworkPolicy]], DAI/DHCP Snooping, [[RPKI]]), Axis B encryption + authentication ([[mTLS]]/[[TLS]]/certificate pinning — neutralizes seizure, the strongest), Axis C detection (iptables-change audit, [[Falco]]). In practice: mTLS between internal services + NetworkPolicy + ban privileged containers.

## MEW 2018 BGP Hijacking

2018-04-24: [[BGP]] hijacking (L3, more-specific advertisement of [[AWS]] [[Route53]] DNS IP range) → [[DNS spoofing]] (myetherwallet.com→attacker IP) → phishing → private-key theft (~215 ETH). Key point: the fake site had no valid [[TLS]] certificate, so the browser warned → only people who ignored the warning got drained. TLS verification is the last line of defense.

## TLS Demo

Three certificates: root [[CA]] (`req -x509`), genuine server ([[CSR]] signed by CA → issuer=MyTrustedRootCA), attacker (self-signed, issuer=bank.local). The domain name can be forged but the CA signature cannot be (no CA private key). [[curl]]: `--resolve` (simulates DNS hijacking), `--cacert` (trust the CA = the defense line). Three scenarios: 1) genuine + trust CA → OK, 2) fake + trust CA → `curl (60) self-signed certificate` connection refused (blocked even when bypassed!), 3) fake + `-k` (verification off) → STOLEN. Hijacking deceives 'where to' (path), but TLS verifies 'is the counterpart real' (identity). In practice, mTLS automates and enforces this.

---

## 한국어

### NAT/DNAT 하이재킹 데모

통찰: NAT 테이블을 쥔 자가 패킷 행선지를 쥔다(root/특권 작업). 데모: 명령어 `iptables -t nat -I DOCKER 1 ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination <B_IP>:80`. 발견: 맥 localhost:8080은 여전히 A([[OrbStack]] docker-proxy가 [[iptables]] 우회). 경로2 VM_IP:8080→B(하이재킹 성공, PREROUTING→DOCKER DNAT 거침). 즉 [[DNAT]] 조작은 그 경로가 iptables를 실제 통과할 때만 트래픽을 가로채다. -D로 정확히 삭제해 복구.

### 네트워크 경로 하이재킹 — 공격과 방어

본질: 경로(라우팅/[[NAT]]/주소해석)를 장악한 자가 트래픽 장악. 공격: 악성 특권 컨테이너 사이드카 [[MITM]], [[Docker]] 소켓 노출, 클라우드 메타데이터 하이재킹, [[ARP]]/[[DHCP]] 스푸핑, [[DNS]]/[[BGP]] 하이재킹, [[Evil Twin]]. 방어: 축A 최소권한(no --privileged, --cap-drop=ALL, K8s [[NetworkPolicy]], DAI/DHCP Snooping, [[RPKI]]), 축B 암호화+인증([[mTLS]]/[[TLS]]/인증서 피닝 — 장악당해도 무력화, 가장 강력), 축C 탐지(iptables 변경 감사, [[Falco]]). 실무: 내부 서비스 간 mTLS + NetworkPolicy + 특권 컨테이너 금지.

### MEW 2018 BGP 하이재킹

2018-04-24: [[BGP]] 하이재킹(L3, [[AWS]] [[Route53]] DNS IP 대역 more-specific 광고) → [[DNS spoofing|DNS 스푸핑]](myetherwallet.com→공격자 IP) → 피싱 → 개인키 탈취(~215 ETH). 핵심: 가짜 사이트는 유효 [[TLS]] 인증서가 없어 브라우저 경고 → 경고 무시한 사람만 털렸다. TLS 검증이 마지막 방어선.

### TLS 데모

인증서 3종: 루트 [[CA]](req -x509), 정품 서버([[CSR]]를 CA가 서명→issuer=MyTrustedRootCA), 공격자(자체서명, issuer=bank.local). 도메인명은 위조 가능하지만 CA 서명은 위조 불가(CA 개인키 없음). [[curl]]: --resolve(DNS 하이재킹 시뮬레이션), --cacert(CA 신뢰=방어선). 세 시나리오: 1)정품+CA신뢰→OK, 2)가짜+CA신뢰→curl (60) self-signed certificate 연결 거부(우회당해도 차단!), 3)가짜+-k(검증 끔)→STOLEN. 하이재킹은 '어디로'(경로) 속이지만 TLS는 '상대가 진짜인지'(신원)를 검증. 실무에선 mTLS로 자동·강제화.
