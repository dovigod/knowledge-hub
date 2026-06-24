---
id: 019ea14b-62e9-7357-b101-a85bf056f8a1
name: mTLS
aliases:
  - mTLS
  - mutual TLS
  - mutual authentication
  - 상호 TLS
updated_at: '2026-06-07T08:55:32.841Z'
summary: >-
  Mutual TLS — both client and server present and verify certificates, used to
  authenticate service-to-service traffic in microservices.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# mTLS

## Overview

mTLS (mutual TLS) extends [[TLS]] so both ends authenticate each other with certificates, not just the server. The server demands the client present a valid cert too. It's the workhorse of zero-trust microservice networks: every service-to-service hop verifies cryptographic identity, so a hijacked route ([[BGP hijacking]], [[DNS spoofing]], [[ARP spoofing]], DNAT via [[iptables NAT]]) cannot impersonate a peer.

> [!note] Standard TLS vs mTLS
> - **TLS:** server proves it's the server. Client is anonymous (or proves identity by another means, like a bearer token).
> - **mTLS:** server proves it's the server AND client proves it's the client. Both sides hold certs, both verify.

## Notes

### Why microservices like it
- Identity-per-service: each service has its own cert signed by an internal CA. Stolen cert blast radius is one service.
- No bearer tokens to leak in logs.
- Routing-layer attacks become useless: even if traffic is redirected, the impostor can't present a cert chain that the peer trusts.

### Tooling
- **Service meshes** (Istio, Linkerd, Consul Connect) auto-issue, rotate, and verify certs via sidecars.
- **SPIFFE / SPIRE** for portable workload identity.
- Cloud-native: AWS App Mesh, GCP Anthos Service Mesh.

### Where it sits in the defense map
From the conversation: defense breaks into three axes —
- **A. Prevent path takeover:** least privilege, no `--privileged`, no docker.sock mount, NetworkPolicy.
- **B. Make takeover useless:** [[TLS]] + **mTLS**, certificate pinning, HSTS, DNSSEC. **Strongest axis.**
- **C. Detect changes:** auditd, Falco, immutable infrastructure.

mTLS is the highest-ROI control in axis B for internal traffic — best cost/benefit in a microservices environment.

> [!tip] Practical combo for K8s
> internal service-to-service [[mTLS]] (via a mesh) + Kubernetes NetworkPolicy + ban privileged containers. Covers the realistic attack surface without slowing teams down.

---

## 한국어

### 개요

mTLS(mutual TLS)는 [[TLS]]를 확장해서 서버뿐 아니라 클라이언트도 인증서로 자기를 증명하게 한다. 서버가 클라이언트에게도 유효한 인증서를 요구. 제로 트러스트 마이크로서비스 네트워크의 핵심 도구 — 모든 서비스 간 홉이 암호학적 신원을 검증하므로, 하이재킹된 경로([[BGP hijacking]], [[DNS spoofing]], [[ARP spoofing]], [[iptables NAT]]의 DNAT)도 상대를 사칭할 수 없다.

> [!note] 표준 TLS vs mTLS
> - **TLS:** 서버가 자기가 서버임을 증명. 클라이언트는 익명(또는 베어러 토큰 등 다른 수단으로 증명).
> - **mTLS:** 서버가 자기를 증명 AND 클라이언트도 자기를 증명. 양쪽 다 인증서 보유, 양쪽 다 검증.

### 노트

#### 마이크로서비스가 좋아하는 이유
- 서비스별 신원: 각 서비스가 내부 CA가 서명한 자기 인증서. 인증서 탈취 피해 반경이 한 서비스로 한정.
- 로그에 새는 베어러 토큰 없음.
- 라우팅 계층 공격이 무용지물: 트래픽이 우회돼도 사칭자는 상대가 신뢰하는 인증서 체인을 제시 못 함.

#### 도구
- **서비스 메시** (Istio, Linkerd, Consul Connect): 사이드카를 통해 인증서를 자동 발급/회전/검증.
- **SPIFFE / SPIRE**: 휴대 가능한 워크로드 신원.
- 클라우드 네이티브: AWS App Mesh, GCP Anthos Service Mesh.

#### 방어 지도에서의 위치
대화에서 정리된 세 축 —
- **A. 경로 장악 자체 막기:** 최소 권한, `--privileged` 금지, docker.sock 마운트 금지, NetworkPolicy.
- **B. 장악당해도 무용지물로:** [[TLS]] + **mTLS**, 인증서 피닝, HSTS, DNSSEC. **가장 강한 축.**
- **C. 변경 탐지:** auditd, Falco, 불변 인프라.

mTLS는 축 B에서 내부 트래픽에 대한 가장 ROI 높은 통제 — 마이크로서비스 환경의 가성비 갑.

> [!tip] K8s 실무 콤보
> 내부 서비스 간 [[mTLS]] (메시 사용) + Kubernetes NetworkPolicy + 특권 컨테이너 금지. 팀 속도 안 떨어뜨리고 현실적 공격면을 커버.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
