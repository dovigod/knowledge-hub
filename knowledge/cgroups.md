---
id: 019ea148-5c06-7381-95de-025476241920
name: cgroups
aliases:
  - cgroup
  - control groups
  - 씨그룹
updated_at: '2026-06-07T08:52:14.470Z'
summary: >-
  Linux kernel feature that limits and accounts for resource usage (CPU, memory,
  I/O, network bandwidth) per process group.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# cgroups

## Overview

cgroups (control groups) are the Linux kernel feature that limits **how much** of a resource a process group can use — CPU, memory, disk I/O, network bandwidth. Together with [[Linux namespaces]] (which isolate visibility), cgroups form the foundation of [[Docker]]-style containers.

> [!note] One-line rule
> [[Linux namespaces]] = visibility isolation (시야). cgroups = **resource isolation** (자원).

## Notes

### Why we need them
namespaces alone are not enough: one container could eat all the host's CPU/memory and starve every other container. cgroups put a hard ceiling on each group.

### Common limits
```
docker run --memory="512m" --cpus="1.5" my-image
```
- `--memory="512m"`: hard cap at 512 MB. Exceed it → OOM kill.
- `--cpus="1.5"`: at most 1.5 CPU cores worth of time.

### Apartment analogy
If [[Linux namespaces]] are the walls between apartments, cgroups are the circuit breaker (두꺼비집) for each one — you only get so much electricity, no matter how greedy your appliances are.

> [!tip] Not network-related
> cgroups have nothing to do with networking or the [[OSI 7-layer model]]. They're pure resource accounting.

---

## 한국어

### 개요

cgroups (control groups)는 프로세스 그룹이 **얼마나 많은** 자원을 쓸 수 있는지 제한·집계하는 Linux 커널 기능이다 — CPU, 메모리, 디스크 I/O, 네트워크 대역폭. [[Linux namespaces]](시야 격리)와 함께 [[Docker]] 식 컨테이너의 토대를 이룬다.

> [!note] 한 줄 요약
> [[Linux namespaces]] = 시야 격리. cgroups = **자원 격리**.

### 노트

#### 왜 필요한가
namespaces만으로는 부족하다 — 한 컨테이너가 호스트의 CPU/메모리를 다 먹어버려 다른 컨테이너가 굶을 수 있다. cgroups는 각 그룹에 단단한 상한을 건다.

#### 흔한 제한
```
docker run --memory="512m" --cpus="1.5" my-image
```
- `--memory="512m"`: 512 MB 하드 캡. 초과 시 OOM kill.
- `--cpus="1.5"`: 최대 1.5 CPU 코어 분량 시간.

#### 아파트 비유
[[Linux namespaces]]가 아파트 사이 벽이라면, cgroups는 각 집의 두꺼비집 — 가전이 아무리 욕심내도 정해진 전기 이상은 못 쓴다.

> [!tip] 네트워크와 무관
> cgroups는 네트워킹이나 [[OSI 7-layer model]]과 관계없다. 순수 자원 회계.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
