---
id: 019e8d53-f187-76c9-ace4-2806e0147005
name: Cilium
aliases:
  - Cilium
  - cilium
updated_at: '2026-06-07T08:04:55.699Z'
summary: >-
  eBPF-based networking, observability, and security solution for cloud-native
  and Kubernetes environments.
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Cilium

## Overview
Cilium is an open-source, cloud-native networking, observability, and security solution built on top of eBPF (extended Berkeley Packet Filter). It is primarily used as a CNI (Container Network Interface) plugin for [[Kubernetes]].

> [!note] Why Cilium
> Unlike traditional CNIs that rely on iptables and IP-based rules, Cilium uses eBPF programs attached to kernel hooks for identity-aware policy enforcement at L3/L4/L7 — with lower overhead than sidecar-based meshes.

## Notes
- Built on eBPF technology for kernel-level programmability
- Provides L3/L4/L7 network policies
- Replaces kube-proxy with eBPF-based service routing
- Offers Hubble for network observability
- Supports service mesh capabilities without sidecars
- Identity-based security model (not IP-based)
- CNCF graduated project

## Security Layers

### L3 Security (Network Layer)
- **Identity-based policy**: Cilium assigns a numeric *security identity* to each pod based on its Kubernetes labels, replacing IP-based ACLs that break under churn.
- **CIDR-based policy**: For external endpoints, allow/deny by CIDR ranges (`toCIDR`, `toCIDRSet`).
- **DNS-based policy** (`toFQDNs`): Resolve fully qualified domain names to allowed IPs at runtime — useful when egress targets are dynamic (e.g., `api.github.com`).
- **Cluster-wide policies** via `CiliumClusterwideNetworkPolicy` for node-level and host-level enforcement.

### L4 Security (Transport Layer)
- Restrict by protocol (`TCP`, `UDP`, `SCTP`) and port.
- Combined with L3 identity to express rules like "pods with label `app=frontend` may reach `app=backend` on TCP/8080 only".
- Enforced in-kernel by eBPF — no userspace proxy hop required.

### L7 Security (Application Layer)
- Deep inspection for **HTTP** (method, path, headers), **gRPC**, **Kafka** (topic/API key), and **DNS**.
- L7 rules are enforced by a transparent **Envoy proxy** spawned per node (not per pod), keeping the sidecar-free model.
- Example: allow `GET /api/v1/products` but deny `POST /api/v1/admin` between two identities.
- mTLS and SPIFFE-based identity issuance supported in newer versions.

> [!tip] Identity over IP
> Because policies bind to identities (label sets), pod rescheduling and IP changes do not invalidate rules — a major win over iptables/CNI policies that must be reprogrammed on every churn event.

## Related
- [[eBPF]] — the kernel technology Cilium is built on
- [[Hubble]] — Cilium's observability layer
- [[Kubernetes]] — primary deployment target
- [[Envoy]] — used for L7 enforcement

---

## 한국어

### 개요
Cilium은 eBPF (extended Berkeley Packet Filter) 기반으로 구축된 오픈소스 클라우드 네이티브 네트워킹, 관측성, 보안 솔루션입니다. 주로 [[Kubernetes]]의 CNI (Container Network Interface) 플러그인으로 사용됩니다.

> [!note] Cilium을 쓰는 이유
> iptables와 IP 기반 규칙에 의존하는 기존 CNI와 달리, Cilium은 커널 훅에 부착된 eBPF 프로그램을 사용하여 L3/L4/L7에서 identity 기반 정책을 시행합니다 — 사이드카 기반 메시보다 오버헤드도 낮습니다.

### 노트
- 커널 레벨 프로그래밍이 가능한 eBPF 기술 기반
- L3/L4/L7 네트워크 정책 제공
- eBPF 기반 서비스 라우팅으로 kube-proxy 대체
- 네트워크 관측성을 위한 Hubble 제공
- 사이드카 없이 서비스 메시 기능 지원
- IP 기반이 아닌 ID 기반 보안 모델
- CNCF graduated 프로젝트

### 보안 계층

#### L3 보안 (네트워크 계층)
- **Identity 기반 정책**: Cilium은 Kubernetes 라벨에 따라 각 Pod에 숫자 *security identity*를 부여하여, Pod 교체로 깨지기 쉬운 IP 기반 ACL을 대체합니다.
- **CIDR 기반 정책**: 외부 엔드포인트에 대해 CIDR 범위 (`toCIDR`, `toCIDRSet`)로 허용/차단합니다.
- **DNS 기반 정책** (`toFQDNs`): FQDN을 런타임에 IP로 해석하여 허용 — egress 대상이 동적일 때 (예: `api.github.com`) 유용합니다.
- **클러스터 전역 정책**: `CiliumClusterwideNetworkPolicy`로 노드 레벨 및 호스트 레벨 시행이 가능합니다.

#### L4 보안 (전송 계층)
- 프로토콜 (`TCP`, `UDP`, `SCTP`)과 포트로 제한합니다.
- L3 identity와 결합하여 "`app=frontend` 라벨의 Pod는 `app=backend`로 TCP/8080만 접근 가능"과 같은 규칙을 표현합니다.
- eBPF가 커널 내에서 시행 — 유저스페이스 프록시 홉이 필요 없습니다.

#### L7 보안 (애플리케이션 계층)
- **HTTP** (메서드, 경로, 헤더), **gRPC**, **Kafka** (토픽/API 키), **DNS**에 대한 심층 검사.
- L7 규칙은 노드당 (Pod당 아님) 투명한 **Envoy 프록시**가 시행하여 사이드카 없는 모델을 유지합니다.
- 예: 두 identity 간에 `GET /api/v1/products`는 허용하지만 `POST /api/v1/admin`은 차단.
- 최신 버전에서는 mTLS와 SPIFFE 기반 identity 발급도 지원합니다.

> [!tip] IP 대신 Identity
> 정책이 identity (라벨 집합)에 바인딩되기 때문에, Pod 재스케줄링이나 IP 변경이 규칙을 무효화하지 않습니다 — 모든 churn 이벤트마다 재프로그래밍이 필요한 iptables/CNI 정책 대비 큰 장점입니다.

### 관련 항목
- [[eBPF]] — Cilium이 기반으로 하는 커널 기술
- [[Hubble]] — Cilium의 관측성 계층
- [[Kubernetes]] — 주된 배포 대상
- [[Envoy]] — L7 시행에 사용됨

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
