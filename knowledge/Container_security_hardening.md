---
id: 019ea14b-9711-750c-a8d9-54f972a4c4e8
name: Container security hardening
aliases:
  - container hardening
  - 컨테이너 보안
  - 컨테이너 하드닝
updated_at: '2026-06-07T08:55:46.193Z'
summary: >-
  Three-axis defense for container environments: prevent path/permission
  takeover, make takeover useless via TLS/mTLS, and detect changes.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Container security hardening

## Overview

Container security hardening organizes defenses against attacks like DNAT hijacking ([[iptables NAT]]), [[ARP spoofing]], [[BGP hijacking]], [[DNS spoofing]], malicious sidecars, and Docker-socket-escape into three complementary axes. The mantra from the source conversation: **whoever controls the route controls the traffic — unless identity verification stops them.**

> [!note] The three axes (memorize these)
> - **A. Prevent path/permission takeover** (least privilege).
> - **B. Make takeover useless** ([[TLS]] / [[mTLS]], encryption, identity). **Strongest axis.**
> - **C. Detect changes** (audit, monitoring, immutability).

## Notes

### Axis A — Prevent takeover
- **Never** run with `--privileged` or `--cap-add=NET_ADMIN` unless absolutely required.
- **Never** mount `/var/run/docker.sock` into untrusted containers (it's full host takeover).
- Default `--cap-drop=ALL` and add back only what's needed.
- `--security-opt=no-new-privileges`.
- Rootless Docker / [[Linux namespaces|user namespaces]].
- Kubernetes **NetworkPolicy** (Calico, Cilium) — pod-to-pod whitelist.
- Stronger sandboxes for untrusted workloads: gVisor, Kata Containers.
- LAN side: DHCP snooping, DAI (against [[ARP spoofing]]), Port Security, 802.1X.
- Internet routing side: RPKI (against [[BGP hijacking]]), DNSSEC (against [[DNS spoofing]]).

### Axis B — Make takeover useless (the killer)
- **End-to-end [[TLS]]** for all traffic.
- **[[mTLS]]** between services (Istio, Linkerd auto-config). Best ROI for microservices.
- Certificate pinning for sensitive clients.
- **HSTS** + DNSSEC on web properties.
- This axis is the moral of [[MyEtherWallet 2018 BGP hijacking]]: even total route + DNS compromise was stopped by certificate validation.

### Axis C — Detect
- **auditd** + **Falco** to alert on iptables/cgroup/syscall anomalies.
- Immutable infrastructure (rebuild rather than patch).
- Image scanning (Trivy, Grype) in CI.
- Runtime drift detection.

### Real-world attacks this defends against
- Malicious sidecar MITM in multi-tenant K8s (via `--privileged`).
- Docker socket mount escape → host takeover.
- Cloud metadata hijacking (`169.254.169.254` IAM credential theft).
- All four L2–L7 redirection families ([[ARP spoofing]], DNAT, [[BGP hijacking]], [[DNS spoofing]]).

> [!tip] Practical recommended baseline
> Microservices in K8s: **[[mTLS]]** between services + **NetworkPolicy** + **ban `--privileged`/docker.sock**. That single combo covers most of the realistic attack surface without slowing teams down.

---

## 한국어

### 개요

컨테이너 보안 하드닝은 DNAT 하이재킹([[iptables NAT]]), [[ARP spoofing]], [[BGP hijacking]], [[DNS spoofing]], 악성 사이드카, Docker 소켓 탈출 같은 공격에 대한 방어를 세 가지 상호보완 축으로 정리한다. 대화의 만트라: **경로를 쥔 자가 트래픽을 쥔다 — 신원 검증이 막지 않는 한.**

> [!note] 세 축 (꼭 외울 것)
> - **A. 경로/권한 장악 자체 막기** (최소 권한).
> - **B. 장악당해도 무용지물로** ([[TLS]] / [[mTLS]], 암호화, 신원). **가장 강한 축.**
> - **C. 변경 탐지** (감사, 모니터링, 불변성).

### 노트

#### 축 A — 장악 자체 막기
- 꼭 필요한 게 아니라면 **절대** `--privileged`나 `--cap-add=NET_ADMIN` 금지.
- 신뢰할 수 없는 컨테이너에 `/var/run/docker.sock` 마운트 **절대** 금지 (호스트 완전 장악).
- 기본 `--cap-drop=ALL`, 필요한 것만 다시 부여.
- `--security-opt=no-new-privileges`.
- Rootless Docker / [[Linux namespaces|user namespace]].
- Kubernetes **NetworkPolicy** (Calico, Cilium) — 파드 간 화이트리스트.
- 신뢰할 수 없는 워크로드는 더 강한 샌드박스: gVisor, Kata Containers.
- LAN 측: DHCP snooping, DAI ([[ARP spoofing]] 대응), Port Security, 802.1X.
- 인터넷 라우팅 측: RPKI ([[BGP hijacking]] 대응), DNSSEC ([[DNS spoofing]] 대응).

#### 축 B — 장악당해도 무용지물 (결정타)
- 모든 트래픽에 **종단간 [[TLS]]**.
- 서비스 간 **[[mTLS]]** (Istio, Linkerd가 자동 구성). 마이크로서비스 ROI 최고.
- 민감한 클라이언트엔 인증서 피닝.
- 웹 자산엔 **HSTS** + DNSSEC.
- 이 축은 [[MyEtherWallet 2018 BGP hijacking]]의 교훈 — 경로+DNS가 완전히 뚫려도 인증서 검증이 막았다.

#### 축 C — 탐지
- **auditd** + **Falco**로 iptables/cgroup/syscall 이상 알림.
- 불변 인프라(패치보다 재빌드).
- CI에서 이미지 스캐닝(Trivy, Grype).
- 런타임 드리프트 탐지.

#### 이게 막는 실제 공격
- 멀티테넌트 K8s에서 악성 사이드카 MITM (`--privileged` 통한).
- Docker 소켓 마운트 탈출 → 호스트 장악.
- 클라우드 메타데이터 하이재킹(`169.254.169.254` IAM 자격증명 탈취).
- L2~L7 우회 공격 가족 네 가지 모두 ([[ARP spoofing]], DNAT, [[BGP hijacking]], [[DNS spoofing]]).

> [!tip] 실무 권장 베이스라인
> K8s 마이크로서비스: 서비스 간 **[[mTLS]]** + **NetworkPolicy** + **`--privileged`/docker.sock 금지**. 이 한 콤보가 팀 속도를 안 떨어뜨리면서 현실적 공격면 대부분을 커버한다.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
