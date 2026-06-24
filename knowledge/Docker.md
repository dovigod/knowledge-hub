---
id: 019ea148-5bfd-71ff-8e3a-84039f6cb3ee
name: Docker
aliases:
  - docker
  - 도커
updated_at: '2026-06-07T08:52:14.461Z'
summary: >-
  Container platform that packages applications with their dependencies into
  portable, isolated runtime units using Linux kernel primitives.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Docker

## Overview

Docker is a container platform that packages an application together with everything it needs to run (libraries, config, runtime) into an immutable image that can run anywhere a Docker engine exists. It did not invent isolation — it wraps Linux kernel features ([[namespaces]] and [[cgroups]]) into an approachable workflow.

> [!note] Three-part mental model
> - **[[Dockerfile]]** = the recipe
> - **Image** = the immutable snapshot (the mold)
> - **Container** = a running instance of the image (the object from the class)

## Notes

### History
- 2000s: VMware-era VMs gave isolation but were heavy (each VM ships a whole OS, GBs, tens of seconds to boot).
- 2008: Linux kernel adds [[cgroups]] + [[namespaces]].
- 2013: Solomon Hykes (dotCloud) releases Docker, making those kernel features easy.
- 2015: OCI standard formalized; Kubernetes emerges.
- Today: containers are the de-facto deployment unit.

### vs VM
- VM: full guest OS per instance, slow boot, GBs.
- Container: shares host kernel, only app + libs isolated, sub-second start, tens to hundreds of MB.

### Why teams use it
- Environment consistency ("works on my machine" disease cure)
- Clean install/uninstall (just remove the container)
- Parity between dev and prod ("build once, run anywhere")
- Spin up infra (redis/postgres/rabbitmq) with one command

> [!tip] Mac caveat
> macOS has no Linux kernel. [[OrbStack]] / Docker Desktop run a small Linux VM under the hood. `docker version` shows Client=darwin, Server=linux. Mac-side `iptables` / `ip addr` won't show container plumbing — get inside the VM or the container to see real Linux.

---

## 한국어

### 개요

Docker는 애플리케이션과 그 실행에 필요한 모든 것(라이브러리, 설정, 런타임)을 불변 이미지로 묶어서, Docker 엔진이 있는 곳이라면 어디서든 동일하게 실행할 수 있게 하는 컨테이너 플랫폼이다. 격리 기술을 새로 발명한 것이 아니라, Linux 커널의 [[namespaces]]와 [[cgroups]]를 쉽게 쓰도록 포장한 것이다.

> [!note] 3요소 멘탈 모델
> - **[[Dockerfile]]** = 레시피
> - **Image** = 불변 스냅샷 (붕어빵 틀)
> - **Container** = 이미지를 실행한 인스턴스 (틀로 찍어낸 붕어빵)

### 노트

#### 역사
- 2000년대: VMware 시대의 VM은 격리는 됐지만 무거움(VM마다 OS 전체, 수 GB, 부팅 수십 초).
- 2008년: Linux 커널에 [[cgroups]] + [[namespaces]] 추가.
- 2013년: Solomon Hykes(dotCloud)가 Docker 발표, 커널 기능을 쉽게 쓰게 함.
- 2015년: OCI 표준 제정, Kubernetes 등장.
- 현재: 컨테이너가 사실상 표준 배포 단위.

#### VM과 비교
- VM: 인스턴스마다 게스트 OS 전체, 부팅 느림, 수 GB.
- 컨테이너: 호스트 커널 공유, 앱+라이브러리만 격리, 1초 이내 시작, 수십~수백 MB.

#### 왜 쓰는가
- 환경 일관성 ("내 컴퓨터에선 됐는데" 병 치료)
- 깨끗한 설치/제거 (컨테이너 지우면 끝)
- 개발/프로덕션 동일성 ("build once, run anywhere")
- 인프라(redis/postgres/rabbitmq) 한 번에 띄우기

> [!tip] 맥 주의사항
> macOS에는 Linux 커널이 없다. [[OrbStack]] / Docker Desktop은 내부에 작은 Linux VM을 띄운다. `docker version`에서 Client=darwin, Server=linux. 맥 쪽에서 `iptables`/`ip addr` 해도 컨테이너 배선 안 보임 — VM이나 컨테이너 안에 들어가야 진짜 Linux가 보인다.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
