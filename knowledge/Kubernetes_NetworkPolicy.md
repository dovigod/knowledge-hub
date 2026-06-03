---
id: 019e8d54-3173-751c-83ed-5f8700535d68
name: Kubernetes NetworkPolicy
aliases:
  - K8s NetworkPolicy
  - NetworkPolicy
updated_at: '2026-06-03T11:52:45.683Z'
summary: >-
  Kubernetes resource that controls pod-to-pod and pod-to-external traffic at
  L3/L4 using label selectors.
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Kubernetes NetworkPolicy

## Overview
Kubernetes NetworkPolicy is a native Kubernetes resource that specifies how groups of pods are allowed to communicate with each other and other network endpoints, operating at L3/L4.

## Notes
- Uses label selectors to identify pods
- Operates at L3 (IP) and L4 (ports/protocols)
- Requires a CNI plugin that supports NetworkPolicy (Cilium, Calico, Weave)
- Default behavior: all traffic allowed until a policy applies
- Supports ingress and egress rules
- Cannot enforce L7 policies natively — needs Cilium or service mesh for that

---

## 한국어

### 개요
Kubernetes NetworkPolicy는 파드 그룹이 서로 및 다른 네트워크 엔드포인트와 통신하는 방식을 지정하는 Kubernetes 네이티브 리소스로, L3/L4에서 동작합니다.

### 노트
- 라벨 셀렉터로 파드 식별
- L3(IP)와 L4(포트/프로토콜)에서 동작
- NetworkPolicy를 지원하는 CNI 플러그인 필요 (Cilium, Calico, Weave)
- 기본 동작: 정책이 적용되기 전까지 모든 트래픽 허용
- ingress와 egress 규칙 지원
- L7 정책은 네이티브로 적용 불가 — Cilium이나 서비스 메시 필요

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
