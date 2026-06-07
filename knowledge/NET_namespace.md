---
id: 019ea148-710b-76bf-a037-ffe5bfae7751
name: NET namespace
aliases:
  - NET ns
  - netns
  - network namespace
updated_at: '2026-06-07T08:52:19.851Z'
summary: >-
  Linux network namespace giving each container an isolated copy of the entire
  network stack — interfaces, IPs, port table, routing, iptables.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# NET namespace

## Overview

A NET namespace is one of the [[Linux namespaces]] types: it gives a process group **its own copy of the entire network stack** — interfaces, IPs, port table, routing table, firewall rules. Two containers can both bind port 3000 without conflict because each has its own port table, not because the kernel rewrites ports.

> [!note] The crucial misconception
> Containers do NOT get different ports assigned silently. Both really use 3000. The reason there's no clash: each lives in a separate NET namespace, so it's the "3000 of world A" vs the "3000 of world B." Like apartment 101 room 1 vs apartment 102 room 1.

## Notes

### What's in a network stack (and therefore duplicated per namespace)
- Network interfaces (`eth0`, `lo`)
- IP addresses
- **Port table** ("who owns which port")
- Routing table
- Firewall rules ([[iptables NAT|iptables]])

Normally Linux has one of these sets shared by every process. Creating a NET namespace clones the whole set.

### What actually happens when an app calls `listen(3000)`
1. App calls `bind(socket, port=3000)`.
2. Kernel checks which NET namespace the process belongs to.
3. Kernel checks **that namespace's** port table.
4. If free → record "port 3000 owned by this PID" in that namespace's book.
5. If taken → `EADDRINUSE`.

The kernel never compares against other namespaces' port tables. They are mutually invisible.

### Talking to the outside world
A fresh NET namespace is an isolated island — zero links to anything else. To connect it to the host network you need plumbing:
- [[veth pair]] — virtual cable, one end in the container, the other on the host.
- [[docker0 bridge]] — a virtual switch on the host that those host-side ends plug into.
- [[iptables NAT]] — translates addresses for inbound (`-p` / DNAT) and outbound (MASQUERADE) traffic.

> [!example] Port collision math
> 100 services × port 3000, all in distinct NET namespaces = zero internal conflicts. Only when you publish them to the host (`-p`) does the **host** port have to be unique.

---

## 한국어

### 개요

NET namespace는 [[Linux namespaces]]의 한 종류로, 프로세스 그룹에게 **네트워크 스택 전체를 통째로 복사한 자기만의 사본**을 준다 — 인터페이스, IP, 포트 테이블, 라우팅 테이블, 방화벽 규칙. 두 컨테이너가 동시에 3000번 포트에 bind해도 충돌하지 않는 이유는 커널이 포트를 바꿔치기해서가 아니라, 각자 자기 포트 테이블을 갖고 있기 때문이다.

> [!note] 핵심 오해 바로잡기
> 컨테이너가 몰래 다른 포트를 할당받는 게 아니다. 둘 다 진짜로 3000번을 쓴다. 충돌이 안 나는 이유: 각각 다른 NET namespace에 살아서, "A 세상의 3000"과 "B 세상의 3000"이기 때문. 101호의 1번 방과 102호의 1번 방 같은 것.

### 노트

#### 네트워크 스택에 들어있는 것 (namespace마다 복제됨)
- 네트워크 인터페이스 (`eth0`, `lo`)
- IP 주소
- **포트 테이블** ("어떤 포트를 누가 쓰는지")
- 라우팅 테이블
- 방화벽 규칙 ([[iptables NAT|iptables]])

보통 Linux는 이 세트가 하나라 모든 프로세스가 공유한다. NET namespace를 만들면 이 세트 전체를 통째로 새로 복사한다.

#### 앱이 `listen(3000)` 할 때 실제로 일어나는 일
1. 앱이 `bind(socket, port=3000)` 호출.
2. 커널이 이 프로세스가 속한 NET namespace 확인.
3. **그 namespace의** 포트 테이블 확인.
4. 비어있으면 "3000번은 이 PID 것"이라고 그 namespace 장부에 기록.
5. 차있으면 `EADDRINUSE` 에러.

커널은 다른 namespace의 포트 테이블과 절대 비교하지 않는다. 서로의 존재를 모름.

#### 바깥과 통신하는 법
갓 만들어진 NET namespace는 외딴 섬 — 다른 어디와도 연결돼 있지 않다. 호스트 네트워크와 잇으려면 배선이 필요:
- [[veth pair]] — 가상 랜선, 한쪽 끝은 컨테이너 안, 다른 쪽은 호스트.
- [[docker0 bridge]] — 호스트의 가상 스위치, 호스트 쪽 veth 끝이 여기 꽂힘.
- [[iptables NAT]] — 들어오는 트래픽(`-p` / DNAT)과 나가는 트래픽(MASQUERADE)의 주소 번역기.

> [!example] 포트 충돌 계산
> 100개 서비스 × 3000번 포트, 모두 다른 NET namespace = 내부 충돌 0. 호스트로 publish할 때만(`-p`) **호스트** 포트가 유일해야 한다.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
