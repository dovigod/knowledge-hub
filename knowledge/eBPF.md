---
id: 019e8d53-f189-7322-bd9e-8bc4432095a7
name: eBPF
aliases:
  - extended BPF
  - extended Berkeley Packet Filter
updated_at: '2026-06-03T11:52:29.321Z'
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

## Notes
- Runs programs in a sandboxed virtual machine inside the kernel
- Used for networking, observability, security, and performance analysis
- Foundation for tools like Cilium, Falco, Pixie, bpftrace
- Verifier ensures program safety before execution
- JIT compilation for near-native performance
- Hook points: kprobes, uprobes, tracepoints, XDP, tc

---

## 한국어

### 개요
eBPF (extended Berkeley Packet Filter)는 커널 소스 코드를 변경하거나 커널 모듈을 로드하지 않고도 커널 내부에서 안전하고 샌드박스화된 프로그램을 실행할 수 있게 해주는 혁신적인 Linux 커널 기술입니다.

### 노트
- 커널 내부의 샌드박스 가상 머신에서 프로그램 실행
- 네트워킹, 관측성, 보안, 성능 분석에 사용
- Cilium, Falco, Pixie, bpftrace 같은 도구의 기반
- 실행 전 검증기가 프로그램 안전성 보장
- 거의 네이티브에 가까운 성능을 위한 JIT 컴파일
- 훅 포인트: kprobes, uprobes, tracepoints, XDP, tc

## Sources

- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
