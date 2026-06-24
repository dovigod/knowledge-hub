---
id: 019e8d54-3173-751c-83ed-5f8700535d68
name: Kubernetes NetworkPolicy
aliases:
  - K8s Network Policy
  - K8s NetworkPolicy
  - Network Policy
  - NetworkPolicy
updated_at: '2026-06-07T08:05:51.029Z'
summary: >-
  Kubernetes resource that controls pod-to-pod and pod-to-external traffic at
  L3/L4 using label selectors.
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Kubernetes NetworkPolicy

## Overview

> [!note] Kubernetes NetworkPolicy is a native Kubernetes resource that specifies how groups of pods are allowed to communicate with each other and other network endpoints, operating at L3/L4.

It is the standard primitive for [[L3 보안]] and [[L4 보안]] inside a cluster, but relies on a capable [[CNI]] plugin such as [[Cilium]] to enforce the rules.

## Notes
- Uses label selectors to identify pods
- Operates at L3 (IP/CIDR) and L4 (ports/protocols) — the same layers covered by [[L3 보안]] and [[L4 보안]]
- Requires a CNI plugin that supports NetworkPolicy ([[Cilium]], Calico, Weave)
- Default behavior: all traffic allowed until a policy applies
- Supports ingress and egress rules
- Cannot enforce L7 policies natively — needs [[Cilium]] CiliumNetworkPolicy or a service mesh for [[L7 보안]]

> [!warning] NetworkPolicy alone cannot inspect HTTP paths, gRPC methods, or Kafka topics. For application-layer authorization use [[Cilium]] CNP or an L7 proxy.

> [!tip] Pair NetworkPolicy with [[SSH 프로토콜]] hardening and node-level firewalls — pod-level rules don't protect the control plane or node SSH surface.

## Examples
- Deny-all default + explicit allow rules per namespace
- Egress rule restricting pods to a known external CIDR (L3)
- Ingress rule allowing only port 8080 from pods labeled `app=frontend` (L4)
- Upgrade path to [[Cilium]] CiliumNetworkPolicy for HTTP method/path filtering (L7)

---

## 한국어

### 개요

> [!note] Kubernetes NetworkPolicy는 파드 그룹이 서로 및 다른 네트워크 엔드포인트와 통신하는 방식을 지정하는 Kubernetes 네이티브 리소스로, L3/L4에서 동작합니다.

클러스터 내부의 [[L3 보안]]과 [[L4 보안]]을 위한 표준 프리미티브이며, 규칙 시행은 [[Cilium]]과 같은 [[CNI]] 플러그인에 의존합니다.

### 노트
- 라벨 셀렉터로 파드 식별
- L3(IP/CIDR)과 L4(포트/프로토콜)에서 동작 — [[L3 보안]], [[L4 보안]]이 다루는 계층과 동일
- NetworkPolicy를 지원하는 CNI 플러그인 필요 ([[Cilium]], Calico, Weave)
- 기본 동작: 정책이 적용되기 전까지 모든 트래픽 허용
- ingress와 egress 규칙 지원
- L7 정책은 네이티브로 적용 불가 — [[L7 보안]]은 [[Cilium]] CiliumNetworkPolicy나 서비스 메시 필요

> [!warning] NetworkPolicy 단독으로는 HTTP 경로, gRPC 메서드, Kafka 토픽을 검사할 수 없습니다. 애플리케이션 계층 인가는 [[Cilium]] CNP나 L7 프록시를 사용하세요.

> [!tip] NetworkPolicy는 [[SSH 프로토콜]] 강화 및 노드 레벨 방화벽과 함께 운용하세요 — 파드 레벨 규칙은 컨트롤 플레인이나 노드 SSH 표면을 보호하지 않습니다.

### 예시
- 기본 deny-all + 네임스페이스별 명시적 allow 규칙
- 알려진 외부 CIDR로만 파드 egress 제한 (L3)
- `app=frontend` 라벨이 붙은 파드에서 포트 8080만 허용하는 ingress 규칙 (L4)
- HTTP 메서드/경로 필터링을 위해 [[Cilium]] CiliumNetworkPolicy로 업그레이드 (L7)

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
