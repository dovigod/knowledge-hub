---
id: 019ea124-3c8e-730c-b329-ff896042ca3c
title: Docker와 컨테이너 네트워킹
topics:
  - docker
  - container
  - namespaces
  - cgroups
  - veth
  - bridge
  - iptables
  - nat
  - 포트 매핑
  - osi
  - networking
  - infra
sources:
  - 019ea121-0888-7779-b38c-63c0aa2fc563
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
created_at: '2026-06-07T08:12:47.118Z'
updated_at: '2026-06-07T08:32:42.714Z'
---
## Docker — A Complete Guide

### 1. History of Docker

The beginning of the problem — "but it worked on my computer!" Until the 2000s, software deployment was a nightmare because of environment differences (OS versions, library versions, configuration values).

Timeline: ~2000s [[VMware]] and others used virtual machines (VMs) for isolation (heavy and slow) → 2008 [[cgroups]] + [[namespaces]] added to the Linux kernel → 2013 Solomon Hykes of dotCloud announced [[Docker]] (an easy wrapper around kernel features) → 2015 [[OCI]] standard established, [[Kubernetes]] emerges → today containers are the de facto standard for deployment.

Key point: Docker is not the invention of new technology, but a way to make Linux kernel isolation features (cgroups, namespaces) — which were already there — easy for anyone to use.

### 2. How Docker Works

VM vs container: A VM carries its own full OS (several GB) (tens of seconds boot time, heavy). A container shares the host's kernel and isolates only the app + necessary libraries (starts in under a second, tens to hundreds of MB).

Three core elements: [[Dockerfile]] (the recipe that creates an image) → [[Image]] (an immutable snapshot containing everything needed to run) → [[Container]] (an instance of running an image). If image is a class, container is an object. (Dockerfile = recipe, Image = bungeoppang mold, Container = bungeoppang)

Isolation is implemented via two Linux kernel features: [[namespaces]] (a process sees only "its own") and [[cgroups]] (CPU/memory usage limits).

### 3. Direct Local Installation vs Docker

This repo's `yarn up:infra` is an example that brings up redis/postgres/rabbitmq together via docker. Environment consistency, no version conflicts, clean install/removal, fast start, production parity. Core value: "Build once, run anywhere."

### 4. Network Port Mapping

Important: Port mapping is NOT in the Dockerfile. The Dockerfile's `EXPOSE` is just documentation/a memo; actual mapping is done with `docker run -p` or in docker-compose.yml.

A container has its own isolated network space, so internal ports are not visible from outside (a room blocked by walls). Port mapping is punching a window in that wall.

`docker run -p 8080:3000 my-service` → left 8080 = host (MacBook) port, right 3000 = container internal port. Accessing localhost:8080 → connects to the container's 3000-port app. `left:right` = `host:container`.

---

## namespaces and cgroups — The Two Pillars of Container Isolation

namespaces = "what can be seen" (visibility isolation), cgroups = "what and how much can be used" (resource isolation). Apartment analogy: namespaces erect walls between rooms so they can't see each other; cgroups set electricity/water usage limits for each room.

### 1. namespaces — Visibility Isolation

Types: PID (process ID), NET (network), MNT (filesystem/mount), UTS (hostname), USER (user ID), IPC (inter-process communication).

PID namespace example: even if node has PID 5678 on the host, inside the container `ps aux` shows only its own process and it appears as PID 1. NET namespace example: each container has its own network stack, so both using port 3000 does not conflict. namespaces are a room surrounded by one-way mirrors.

### 2. cgroups — Resource Limits

If only visibility is isolated, one container can hog resources and starve others. cgroups limit CPU/memory/disk I/O/network bandwidth. Example: `docker run --memory="512m" --cpus="1.5"`. On memory overflow, OOM kill. cgroups are the breaker box (circuit breaker) of each room.

### 3. Combining the Two = Container

namespaces (visibility isolation) + cgroups (resource isolation) = an isolated, restricted, independent execution environment. Docker is the tool that automatically and easily bundles the two.

---

## Clarification: Same Port 3000, No Conflict

Exactly one thing is backwards. Misunderstanding: it's not "because different ports are assigned" — the two containers really do use the same 3000. It's not that ports are swapped; both use 3000 yet do not conflict. The reason: each container has its own NET namespace, so even the same "3000" is 3000 in different worlds. Like Apartment 101 Room 1 and Apartment 102 Room 1.

PID and port are separate: PID namespace (process number isolation) and NET namespace (network/port isolation) are different namespaces. It's "ports are fine because the NET namespace is separate."

Conflict does not happen inside the container, but is possible when bringing ports out to the host (MacBook). So we use only different host numbers like `user-service ports ["3000:3000"]`, `clutch-service ports ["3010:3000"]`.

---

## The Identity of NET namespace and Actual Port Allocation

NET namespace = a whole copy of the network stack (interfaces, IP, port table, routing table, iptables). The port table is separate per namespace, so each can write "3000 in use" in its own ledger without conflict.

Actual port allocation: `app.listen(3000)` → `bind(socket, port=3000)` system call → the kernel checks that NET namespace's port ledger → if free, records it; if occupied, EADDRINUSE. Port allocation = the kernel writing "this number = this process" in that namespace's ledger.

Connecting from outside: [[veth pair]] (a virtual cable connecting two namespaces, one end = container `eth0` / the other end = host `vethXXX`), [[docker0]] (virtual switch), [[iptables]] DNAT (the `-p 3010:3000` flag creates a rule "host:3010 → containerIP:3000").

---

## The Mac Trap and Live Demo

The Mac trap: there is no Linux on a Mac. Docker spins up a small Linux VM inside the Mac and runs containers in it. In `docker version`, Client = darwin/arm64, Server = linux/arm64. iptables/ip addr don't appear on the Mac host (they're inside the VM). Inside the container it's real Linux.

Demo (two nginx containers, internal 80, hosts 8001/8002): both containers LISTEN on internal 80 (no conflict), `ps aux` shows only nginx as PID 1, `eth0@if13` + IP, from Mac 8001 → .2 / 8002 → .3 reachable. Cheatsheet: Mac (`docker ps`, `inspect`, `curl`, `docker exec -it sh`), inside the container (`ps aux`, `ip addr`, `netstat`), VM itself (enter via `orb` then `iptables -t nat -L`).

---

## veth pair, docker0 bridge, NAT

veth pair = a magic cable with both ends open. Data that enters one end comes out the other. End A = container `eth0`, End B = host `vethXXXX`. The `@if13` in `eth0@if13` = the interface number of the other end. docker0 bridge = a virtual switch where multiple cables plug in; provides container-to-container communication + gateway.

Two directions: outgoing traffic (automatic, masquerade NAT changes the source to the host IP), incoming traffic (requires `-p`, DNAT changes the destination to containerIP:port). Key: veth + bridge = wiring, iptables NAT = address translation.

---

## Live veth Demo

Live veth demo (`nginx -p 8080:80`): inside the container `5: eth0@if17`, on the VM side `17: veth6ec1993@if5 ... master docker0` → both ends point to each other (one cable strand). docker0 IP = 192.168.215.1 = gateway. iptables: incoming path DNAT (`--dport 8080 → 192.168.215.2:80`), outgoing path MASQUERADE. `curl localhost:8080` → HTTP 200.

---

## Mapping to the OSI Layers

L7 nginx/HTTP/curl, L4 ports 80/8080/`bind()`/TCP LISTEN, L3 IP/iptables NAT (DNAT/MASQUERADE), L2 veth/docker0/MAC, L1 veth virtual cable. veth = virtual NIC (L2, frames with MAC), docker0 = virtual switch (L2, also has an IP so it doubles as an L3 gateway), iptables NAT = L3 (IP) + L4 (port). Outside OSI: PID namespace, cgroups (OS-level isolation). NET namespace = the container that wholesale-clones the L2~L4 stack.

---

## 한국어

### 1. Docker의 역사

문제의 시작 — '내 컴퓨터에선 됐는데?' 2000년대까지 소프트웨어 배포는 환경 차이(OS 버전, 라이브러리 버전, 설정값) 때문에 악몽이었다.

연대기: ~2000년대 [[VMware]] 등 가상 머신(VM)으로 격리(무겁고 느림) → 2008 Linux 커널에 [[cgroups]] + [[namespaces]] 추가 → 2013 dotCloud의 Solomon Hykes가 [[Docker]] 발표(커널 기능을 쉽게 포장) → 2015 [[OCI]] 표준 제정, [[Kubernetes]] 등장 → 현재 컨테이너가 배포의 사실상 표준.

핵심: Docker는 새 기술 발명이 아니라 Linux 커널에 이미 있던 격리 기술(cgroups, namespaces)을 누구나 쉽게 쓰게 만든 것.

### 2. Docker는 어떻게 작동하는가

VM vs 컨테이너: VM은 각자 OS 전체(수 GB)를 가짐(부팅 수십 초, 무거움). 컨테이너는 호스트의 커널을 공유하고 앱+필요한 라이브러리만 격리(시작 1초 이내, 수십~수백 MB).

핵심 3요소: [[Dockerfile]](이미지를 만드는 레시피) → [[Image]](실행에 필요한 모든 것을 담은 불변 스냅샷) → [[Container]](이미지를 실행한 인스턴스). 이미지가 클래스라면 컨테이너는 객체. (Dockerfile=레시피, Image=붕어빵 틀, Container=붕어빵)

격리는 Linux 커널의 두 기능으로: [[namespaces]](프로세스가 '내 것'만 보게 함), [[cgroups]](CPU/메모리 사용량 제한).

### 3. 로컬 직접 설치 vs Docker

이 레포의 `yarn up:infra`가 redis/postgres/rabbitmq를 docker로 한 번에 띄우는 예시. 환경 일관성, 버전 충돌 없음, 깨끗한 설치/삭제, 빠른 시작, 프로덕션 동일성. 핵심 가치: 'Build once, run anywhere'.

### 4. 네트워크 포트 매핑

중요: 포트 매핑은 Dockerfile에 없다. Dockerfile의 EXPOSE는 단순 문서/메모일 뿐, 실제 매핑은 `docker run -p` 또는 docker-compose.yml에서 한다.

컨테이너는 격리된 자기만의 네트워크 공간을 가져서 외부에서 내부 포트가 안 보인다(벽으로 막힌 방). 포트 매핑은 그 벽에 창문을 뚫는 것.

`docker run -p 8080:3000 my-service` → 왼쪽 8080=호스트(맥북) 포트, 오른쪽 3000=컨테이너 내부 포트. localhost:8080 접속 → 컨테이너의 3000번 앱으로 연결. `왼쪽:오른쪽` = `호스트:컨테이너`.

---

### namespaces와 cgroups — 컨테이너 격리의 두 기둥

namespaces = '무엇을 볼 수 있는가'(시야 격리), cgroups = '무엇을 얼마나 쓸 수 있는가'(자원 격리). 아파트 비유: namespaces는 방마다 벽을 세워 서로 못 보게, cgroups는 각 방에 전기/수도 사용량 한도.

#### 1. namespaces — 시야 격리

종류: PID(프로세스 ID), NET(네트워크), MNT(파일시스템/마운트), UTS(호스트명), USER(사용자 ID), IPC(프로세스 간 통신).

PID namespace 예시: 호스트에서 node가 PID 5678이어도 컨테이너 안에서 ps aux 하면 자기 프로세스만 보이고 PID 1로 보인다. NET namespace 예시: 각 컨테이너가 자기만의 네트워크 스택을 가져서 둘 다 3000번 포트를 써도 충돌하지 않는다. namespaces는 일방향 거울로 둘러싸인 방.

#### 2. cgroups — 자원 한도

시야만 격리하면 한 컨테이너가 자원을 다 먹어 다른 컨테이너가 굶을 수 있다. cgroups가 CPU/메모리/디스크 I/O/네트워크 대역폭을 제한. 예: `docker run --memory="512m" --cpus="1.5"`. 메모리 초과 시 OOM kill. cgroups는 각 방의 두꺼비집(전기 차단기).

#### 3. 둘을 합치면 = 컨테이너

namespaces(시야 격리) + cgroups(자원 격리) = 격리되고 제한된 독립 실행 환경. Docker는 둘을 자동으로 쉽게 묶어주는 도구.

---

### 동일 포트 3000, 충돌 없음에 대한 정정

딱 한 군데가 거꾸로다. 오해: '다른 포트를 할당할 테니'가 아니라, 두 컨테이너가 진짜로 똑같이 3000번을 쓴다. 포트를 바꿔치기하는 게 아니라 둘 다 3000번을 쓰는데도 충돌이 안 난다. 이유는 각 컨테이너가 자기만의 NET namespace를 가져서 같은 '3000번'이라도 서로 다른 세상의 3000번이기 때문. 아파트 101호 1번 방과 102호 1번 방 같은 것.

PID와 포트는 별개: PID namespace(프로세스 번호 격리)와 NET namespace(네트워크/포트 격리)는 다른 namespace. 'NET namespace가 따로라서 포트가 괜찮은' 것.

충돌은 컨테이너 안에서는 안 나지만 호스트(맥북) 바깥으로 포트를 꺼낼 때 가능. 그래서 user-service ports ["3000:3000"], clutch-service ports ["3010:3000"]처럼 호스트 번호만 다르게.

---

### NET namespace의 정체와 실제 포트 할당

NET namespace = 네트워크 스택 통째로 복사본(인터페이스, IP, 포트 테이블, 라우팅 테이블, iptables). 포트 테이블이 namespace마다 따로라 각자 자기 장부에 '3000 사용 중'이라 적어도 충돌 없음.

실제 포트 할당: app.listen(3000) → bind(socket, port=3000) 시스템 콜 → 커널이 해당 NET namespace의 포트 장부 확인 → 비어있으면 기록, 차있으면 EADDRINUSE. 포트 할당 = 커널이 그 namespace의 장부에 '이 번호=이 프로세스'라고 적는 행위.

바깥에서 접속: [[veth pair]](두 namespace 잇는 가상 랜선, 한쪽=컨테이너 eth0/다른쪽=호스트 vethXXX), [[docker0]](가상 스위치), [[iptables]] DNAT(`-p 3010:3000`이 생성하는 '호스트:3010 → 컨테이너IP:3000' 규칙).

---

### 맥의 함정과 라이브 데모

맥의 함정: 맥에는 Linux가 없다. Docker는 맥 안에 작은 Linux VM을 띄우고 컨테이너를 그 안에서 돌린다. `docker version`에서 Client=darwin/arm64, Server=linux/arm64. 맥 호스트에선 iptables/ip addr 안 나옴(VM 안에 있음). 컨테이너 안은 진짜 Linux.

데모(nginx 2개, 내부 80, 호스트 8001/8002): 두 컨테이너 모두 내부 80 LISTEN(충돌 없음), ps aux에 nginx만 PID 1, eth0@if13 + IP, 맥에서 8001→.2/8002→.3 도달. 치트시트: 맥(docker ps, inspect, curl, docker exec -it sh), 컨테이너 안(ps aux, ip addr, netstat), VM 자체(orb 진입 후 iptables -t nat -L).

---

### veth pair, docker0 bridge, NAT

veth pair = 양 끝이 뚫린 마법의 랜선. 한쪽 끝에 들어간 데이터는 반대쪽으로 나온다. 끝A=컨테이너 eth0, 끝B=호스트 vethXXXX. eth0@if13의 @if13=반대쪽 끝의 인터페이스 번호. docker0 bridge=여러 랜선 꽂는 가상 스위치, 컨테이너끼리 통신+게이트웨이.

두 방향: 나가는 트래픽(자동, masquerade NAT으로 출발지를 호스트 IP로 변경), 들어오는 트래픽(-p 필요, DNAT으로 목적지를 컨테이너IP:포트로 변경). 핵심: veth+bridge=배선, iptables NAT=주소 번역.

---

### 라이브 veth 데모

라이브 veth 데모(nginx -p 8080:80): 컨테이너 안 '5: eth0@if17', VM 쪽 '17: veth6ec1993@if5 ... master docker0' → 양 끝이 서로를 가리킴(랜선 한 가닥). docker0 IP=192.168.215.1=게이트웨이. iptables: 들어오는 길 DNAT(--dport 8080 → 192.168.215.2:80), 나가는 길 MASQUERADE. curl localhost:8080 → HTTP 200.

---

### OSI 7계층 매핑

L7 nginx/HTTP/curl, L4 포트 80/8080/bind()/TCP LISTEN, L3 IP/iptables NAT(DNAT/MASQUERADE), L2 veth/docker0/MAC, L1 veth 가상 랜선. veth=가상 NIC(L2, MAC 박힌 프레임), docker0=가상 스위치(L2, IP도 가져 L3 게이트웨이 겸함), iptables NAT=L3(IP)+L4(포트). OSI 밖: PID namespace, cgroups(OS 격리). NET namespace=L2~L4 스택 통째 복제하는 그릇.
