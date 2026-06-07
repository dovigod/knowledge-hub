---
id: 019e8d53-f189-7322-bd9e-8bc4432095a7
name: eBPF
aliases:
  - BPF
  - eBPF
  - extended BPF
  - extended Berkeley Packet Filter
updated_at: '2026-06-07T08:05:26.110Z'
summary: >-
  Linux kernel technology that allows sandboxed programs to run inside the
  kernel without modifying kernel source code.
sources:
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# eBPF

## Overview
eBPF (extended Berkeley Packet Filter) is a revolutionary Linux kernel technology that allows safe, sandboxed programs to run within the kernel without changing kernel source code or loading kernel modules.

> [!note] Why it matters
> eBPF turns the kernel into a programmable platform — networking, observability, and security policies can be injected at runtime with near-native performance.

## Notes
- Runs programs in a sandboxed virtual machine inside the kernel
- Used for networking, observability, security, and performance analysis
- Foundation for tools like [[Cilium]], Falco, Pixie, bpftrace
- Verifier ensures program safety before execution
- JIT compilation for near-native performance
- Hook points: kprobes, uprobes, tracepoints, XDP, tc

## Cilium and Network Security Layers
[[Cilium]] is the flagship eBPF-based networking and security project, providing identity-aware policy enforcement across L3/L4/L7 in Kubernetes and cloud-native environments.

- **L3 (Network layer)**: IP/CIDR-based policies, routing, and identity-based filtering — Cilium replaces traditional iptables with eBPF-driven enforcement at the socket and tc/XDP layers.
- **L4 (Transport layer)**: TCP/UDP port filtering, connection tracking, and service-to-service policies based on Kubernetes identities rather than fragile IP rules.
- **L7 (Application layer)**: Protocol-aware filtering for HTTP, gRPC, Kafka, DNS, etc. — enforced via an eBPF-redirected Envoy sidecar-less proxy, enabling rules like "service A may only call `GET /api/v1/users`".

> [!tip] Identity over IPs
> Cilium attaches a security identity to each pod and enforces policy on identity rather than ephemeral IPs — this is what makes [[Kubernetes]] network policy actually scale.

## Examples
- `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s\n", str(args->filename)); }'` — trace file opens
- `cilium policy import l7-policy.yaml` — apply an L7 HTTP rule
- XDP program attached to a NIC for DDoS filtering before the kernel network stack

---

## 한국어

### 개요
eBPF (extended Berkeley Packet Filter)는 커널 소스 코드를 변경하거나 커널 모듈을 로드하지 않고도 커널 내부에서 안전하고 샌드박스화된 프로그램을 실행할 수 있게 해주는 혁신적인 Linux 커널 기술입니다.

> [!note] 왜 중요한가
> eBPF는 커널을 프로그래밍 가능한 플랫폼으로 바꿔, 네트워킹·관측성·보안 정책을 런타임에 거의 네이티브에 가까운 성능으로 주입할 수 있게 합니다.

### 노트
- 커널 내부의 샌드박스 가상 머신에서 프로그램 실행
- 네트워킹, 관측성, 보안, 성능 분석에 사용
- [[Cilium]], Falco, Pixie, bpftrace 같은 도구의 기반
- 실행 전 검증기가 프로그램 안전성 보장
- 거의 네이티브에 가까운 성능을 위한 JIT 컴파일
- 훅 포인트: kprobes, uprobes, tracepoints, XDP, tc

### Cilium과 네트워크 보안 계층
[[Cilium]]은 eBPF 기반 네트워킹·보안 프로젝트의 대표주자로, Kubernetes 및 클라우드 네이티브 환경에서 L3/L4/L7 전반에 걸친 아이덴티티 기반 정책 시행을 제공합니다.

- **L3 (네트워크 계층)**: IP/CIDR 기반 정책, 라우팅, 아이덴티티 기반 필터링 — Cilium은 전통적인 iptables를 소켓 및 tc/XDP 계층의 eBPF 기반 시행으로 대체합니다.
- **L4 (전송 계층)**: TCP/UDP 포트 필터링, 연결 추적, 그리고 취약한 IP 규칙 대신 Kubernetes 아이덴티티 기반의 서비스 간 정책을 시행합니다.
- **L7 (애플리케이션 계층)**: HTTP, gRPC, Kafka, DNS 등 프로토콜 인지 필터링 — eBPF로 리다이렉트되는 사이드카 없는 Envoy 프록시를 통해 "서비스 A는 `GET /api/v1/users`만 호출 가능" 같은 규칙을 시행합니다.

> [!tip] IP가 아닌 아이덴티티
> Cilium은 각 파드에 보안 아이덴티티를 부여하고, 휘발성 IP가 아닌 아이덴티티 기반으로 정책을 시행합니다 — [[Kubernetes]] 네트워크 정책이 실제로 확장 가능해지는 이유입니다.

### 예시
- `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s\n", str(args->filename)); }'` — 파일 오픈 추적
- `cilium policy import l7-policy.yaml` — L7 HTTP 규칙 적용
- 커널 네트워크 스택 진입 전 DDoS 필터링을 위해 NIC에 부착된 XDP 프로그램

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
