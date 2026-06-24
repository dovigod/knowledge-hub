---
id: 019ea148-5c03-7463-86a7-46a97bfab146
name: Linux namespaces
aliases:
  - linux namespace
  - namespaces
  - 네임스페이스
updated_at: '2026-06-07T08:52:14.467Z'
summary: >-
  Linux kernel feature that gives a process its own isolated view of system
  resources (PIDs, network, mounts, hostname, users, IPC).
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Linux namespaces

## Overview

namespaces are the Linux kernel feature that isolates **what a process can see**. They are one of the two pillars of [[Docker]] containers (the other being [[cgroups]]). A namespace gives a process its own view of a particular resource type — PIDs, network stack, mounts, hostname, etc. — so that processes in different namespaces look like they are running on different machines.

> [!note] One-line rule
> namespaces = **시야 격리 (visibility isolation)** — what you can see. [[cgroups]] = resource isolation — how much you can use.

## Notes

### Types
| Namespace | What it isolates |
|---|---|
| PID | Process IDs (inside, your process can be PID 1) |
| NET | Network stack: interfaces, IPs, ports, routing, iptables |
| MNT | Filesystem mount points |
| UTS | Hostname & domainname |
| USER | User & group IDs |
| IPC | Inter-process communication (semaphores, shm) |

### Examples
- **PID:** host sees node at PID 5678; inside container `ps aux` shows only its own processes and node as PID 1.
- **NET:** two containers can both bind port 3000 with zero conflict because each has its own port table (see [[NET namespace]]).

### Apartment analogy
namespaces are walls between apartments — you can't see your neighbor's stuff. [[cgroups]] are the circuit breakers limiting how much power each apartment can draw.

> [!warning] Not in OSI
> PID namespace is OS-process isolation, not a network concept. Only NET namespace touches networking (it duplicates layers 2–4 of the [[OSI 7-layer model]]).

---

## 한국어

### 개요

namespaces는 프로세스가 **무엇을 볼 수 있는지**를 격리하는 Linux 커널 기능이다. [[Docker]] 컨테이너의 두 기둥 중 하나(다른 하나는 [[cgroups]]). namespace가 다른 프로세스끼리는 마치 서로 다른 머신에서 도는 것처럼 보인다 — PID, 네트워크 스택, 마운트, 호스트명 등 각 자원 유형마다 자기만의 시야를 갖는다.

> [!note] 한 줄 요약
> namespaces = **시야 격리** — 무엇을 볼 수 있는가. [[cgroups]] = 자원 격리 — 얼마나 쓸 수 있는가.

### 노트

#### 종류
| Namespace | 격리하는 것 |
|---|---|
| PID | 프로세스 ID (안에서는 PID 1로 보일 수 있음) |
| NET | 네트워크 스택: 인터페이스, IP, 포트, 라우팅, iptables |
| MNT | 파일시스템 마운트 |
| UTS | 호스트명 & 도메인명 |
| USER | 사용자 & 그룹 ID |
| IPC | 프로세스 간 통신 (세마포어, shm) |

#### 예시
- **PID:** 호스트에서는 node가 PID 5678이지만 컨테이너 안 `ps aux`에는 자기 프로세스들만 보이고 node가 PID 1.
- **NET:** 두 컨테이너 모두 3000번 포트에 bind해도 충돌 없음 — 각자 자기 포트 테이블을 갖기 때문 ([[NET namespace]] 참고).

#### 아파트 비유
namespaces는 아파트 사이의 벽 — 옆집을 못 본다. [[cgroups]]는 각 집의 두꺼비집 — 전기 사용량 제한.

> [!warning] OSI 밖
> PID namespace는 OS 프로세스 격리이지 네트워크 개념이 아니다. NET namespace만 네트워킹과 관련 있고, 그것도 [[OSI 7-layer model]]의 L2~L4를 통째로 복제한다.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
