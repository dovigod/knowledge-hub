---
id: 019ea14b-73d0-728a-986c-3e874c232fe3
name: OrbStack
aliases:
  - orb
  - orbstack
updated_at: '2026-06-07T08:55:37.168Z'
summary: >-
  macOS container runtime that runs a lightweight Linux VM under the hood,
  providing the Linux kernel features Docker needs.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OrbStack

## Overview

OrbStack is a macOS app that runs [[Docker]] (and full Linux machines) on Mac by spinning up a lightweight Linux VM in the background. It exists because macOS has no Linux kernel — and Docker requires Linux primitives like [[Linux namespaces]], [[cgroups]], and [[iptables NAT]]. From the user's terminal, `docker` commands feel native; under the hood they're talking to a daemon inside the VM.

> [!note] What `docker version` reveals
> - **Client:** OS/Arch = `darwin/arm64` (your Mac).
> - **Server:** OS/Arch = `linux/arm64` (the hidden Linux VM).
> Same story for Docker Desktop.

## Notes

### Why a VM at all
Containers are a Linux kernel construct. macOS uses the Darwin/XNU kernel which has none of [[Linux namespaces]] / [[cgroups]] / [[iptables NAT]] / [[docker0 bridge]]. The VM is the only way to host them on Mac.

### Consequences
- `iptables`, `ip addr`, `ip route` on the Mac host show **nothing** related to containers — those live inside the VM.
- Network plumbing ([[veth pair]], [[docker0 bridge]], DNAT) is real inside the VM but invisible from macOS.
- Published ports (`docker run -p 8080:80`) are reachable via `localhost:8080` because OrbStack/Docker Desktop run a **userspace proxy** (docker-proxy) that bypasses iptables on the host → container path.

### Inspecting the VM
- `orb` / `orbctl shell` drops into the OrbStack VM.
- From there: `ip addr`, `sudo iptables -t nat -L -n`, cgroup files all work like a real Linux box.
- Docker Desktop equivalent: the `nsenter1` trick to enter PID 1's namespaces.

### Practical implications for security demos
The userspace proxy means `localhost:8080` traffic on the Mac doesn't traverse iptables — so [[iptables NAT|DNAT hijacking]] experiments only "work" when you hit the VM IP directly, not via `localhost`. See the discussion's hijacking demo.

> [!tip] Mac cheatsheet recap
> - macOS host: `docker ps`, `docker inspect`, `curl localhost:port`, `docker exec -it`.
> - Inside container (Linux): `ip addr show eth0`, `netstat -tlnp`, `hostname`.
> - Inside VM (Linux): `orb` or `orbctl shell`, then full `iptables`/`ip` toolkit.

---

## 한국어

### 개요

OrbStack은 macOS용 앱으로, 백그라운드에 가벼운 Linux VM을 띄워서 맥에서 [[Docker]](와 전체 Linux 머신)를 돌릴 수 있게 해준다. macOS에는 Linux 커널이 없는데 — Docker는 [[Linux namespaces]], [[cgroups]], [[iptables NAT]] 같은 Linux 원시 기능을 필요로 하기 때문에 존재. 사용자 터미널에서 `docker` 명령은 네이티브처럼 느껴지지만 실제론 VM 안의 데몬과 통신.

> [!note] `docker version`이 보여주는 것
> - **Client:** OS/Arch = `darwin/arm64` (맥).
> - **Server:** OS/Arch = `linux/arm64` (숨겨진 Linux VM).
> Docker Desktop도 같은 구조.

### 노트

#### 왜 VM이 있어야 하나
컨테이너는 Linux 커널 산물. macOS는 Darwin/XNU 커널을 쓰며 [[Linux namespaces]] / [[cgroups]] / [[iptables NAT]] / [[docker0 bridge]] 어느 것도 없다. VM이 맥에서 이걸 호스팅하는 유일한 길.

#### 결과
- 맥 호스트에서의 `iptables`, `ip addr`, `ip route`는 컨테이너 관련해선 **아무것도** 안 보여줌 — 모두 VM 안에 있음.
- 네트워크 배선([[veth pair]], [[docker0 bridge]], DNAT)은 VM 안에선 실재하지만 macOS에서는 안 보임.
- publish된 포트(`docker run -p 8080:80`)가 `localhost:8080`으로 접속 가능한 이유: OrbStack/Docker Desktop이 **유저스페이스 프록시**(docker-proxy)를 돌려서 호스트→컨테이너 경로의 iptables를 우회.

#### VM 들여다보기
- `orb` / `orbctl shell`로 OrbStack VM에 진입.
- 거기서 `ip addr`, `sudo iptables -t nat -L -n`, cgroup 파일 모두 진짜 Linux 머신처럼 동작.
- Docker Desktop 동등 기법: `nsenter1` 트릭으로 PID 1의 namespace에 진입.

#### 보안 데모에 미치는 영향
유저스페이스 프록시 때문에 맥의 `localhost:8080` 트래픽은 iptables를 통과하지 않는다 — 그래서 [[iptables NAT|DNAT 하이재킹]] 실험은 `localhost`가 아닌 VM IP 직접 접속에서만 "작동"한다. 대화의 하이재킹 데모 참고.

> [!tip] 맥 치트시트 정리
> - macOS 호스트: `docker ps`, `docker inspect`, `curl localhost:포트`, `docker exec -it`.
> - 컨테이너 안 (Linux): `ip addr show eth0`, `netstat -tlnp`, `hostname`.
> - VM 안 (Linux): `orb` 또는 `orbctl shell`, 그다음 `iptables`/`ip` 도구 풀세트.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
