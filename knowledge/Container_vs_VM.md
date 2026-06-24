---
id: 019ea14b-85ec-7089-800e-629516039d96
name: Container vs VM
aliases:
  - containers vs virtual machines
  - 컨테이너 vs VM
updated_at: '2026-06-07T08:55:41.804Z'
summary: >-
  Containers share the host Linux kernel and isolate only the app and its libs
  (lightweight, fast); VMs boot a full guest OS per instance (heavy, slow).
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Container vs VM

## Overview

The central design tradeoff that [[Docker]] popularized: a container shares the host's Linux kernel and isolates only the application + its libraries via [[Linux namespaces]] and [[cgroups]], while a virtual machine (VM) emulates an entire computer and runs a full guest OS on top of a hypervisor.

> [!note] One-line
> Container = process with walls around it. VM = entire computer with walls around it.

## Notes

### Comparison
| Aspect | VM | Container |
|---|---|---|
| OS | Full guest OS per instance | Shares host kernel |
| Size | GBs | Tens to hundreds of MB |
| Boot time | Tens of seconds | < 1 second |
| Isolation strength | Strong (separate kernel) | Weaker (shared kernel) |
| Overhead | High (hypervisor + guest) | Low (just process bookkeeping) |
| Portability | Across hypervisors | Across any container runtime |

### Why containers won most workloads
- Fast to start → great for elastic scaling.
- Small images → fast to pull and ship.
- Same kernel everywhere → fewer surprises moving between machines.
- Easy to orchestrate (Kubernetes).

### When VMs still make sense
- Multi-tenant isolation where shared kernel is unsafe (e.g. running untrusted code).
- Running non-Linux workloads on a Linux host or vice versa.
- Strict regulatory boundaries.
- That's also why on macOS, [[OrbStack]] / Docker Desktop run a Linux VM in order to host containers at all (since macOS has no Linux kernel).

> [!tip] Both at once
> The Mac case is the obvious one — you're using a VM to host containers. In production, gVisor and Kata Containers blur the line: they wrap containers in lighter-weight VMs to recover stronger isolation.

---

## 한국어

### 개요

[[Docker]]가 대중화시킨 핵심 설계 트레이드오프: 컨테이너는 호스트의 Linux 커널을 공유하고 [[Linux namespaces]]와 [[cgroups]]로 앱+라이브러리만 격리하는 반면, 가상 머신(VM)은 컴퓨터 한 대 전체를 에뮬레이션해서 하이퍼바이저 위에 전체 게스트 OS를 띄운다.

> [!note] 한 줄
> 컨테이너 = 벽 두른 프로세스. VM = 벽 두른 컴퓨터 한 대.

### 노트

#### 비교
| 측면 | VM | 컨테이너 |
|---|---|---|
| OS | 인스턴스마다 전체 게스트 OS | 호스트 커널 공유 |
| 크기 | 수 GB | 수십~수백 MB |
| 부팅 시간 | 수십 초 | 1초 미만 |
| 격리 강도 | 강함 (별개 커널) | 약함 (커널 공유) |
| 오버헤드 | 높음 (하이퍼바이저 + 게스트) | 낮음 (프로세스 회계만) |
| 휴대성 | 하이퍼바이저 호환 | 모든 컨테이너 런타임 호환 |

#### 컨테이너가 대다수 워크로드에서 이긴 이유
- 시작이 빠름 → 탄력적 스케일링에 강함.
- 작은 이미지 → 빠른 pull과 배포.
- 어디서나 같은 커널 → 머신 간 이동의 놀라움 적음.
- 오케스트레이션 쉬움(Kubernetes).

#### 그래도 VM이 의미 있을 때
- 커널 공유가 안전하지 않은 멀티테넌트 격리(예: 신뢰할 수 없는 코드 실행).
- Linux 호스트에서 비-Linux 워크로드 또는 그 반대.
- 엄격한 규제 경계.
- macOS에서 [[OrbStack]] / Docker Desktop이 컨테이너를 호스팅하려고 Linux VM을 돌리는 것도 같은 이유(macOS에는 Linux 커널이 없음).

> [!tip] 둘 다 한 번에
> 맥 케이스는 명백 — 컨테이너 호스팅을 위해 VM을 쓴다. 프로덕션에서는 gVisor와 Kata Containers가 경계를 흐림 — 더 강한 격리를 회복하려고 컨테이너를 가벼운 VM으로 감싼다.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
