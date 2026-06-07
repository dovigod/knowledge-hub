---
id: 019ea148-710d-77ae-a84b-02a6053a8a3f
name: veth pair
aliases:
  - veth
  - virtual ethernet pair
updated_at: '2026-06-07T08:52:19.853Z'
summary: >-
  Pair of virtual Ethernet interfaces connected end-to-end like a magic cable,
  used to bridge a container's NET namespace to the host.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# veth pair

## Overview

A veth pair is a pair of **virtual Ethernet interfaces** that always come in twos and act like a magic cable: anything that goes in one end comes out the other. Containers use them to bridge their isolated [[NET namespace]] to the host's network. One end sits in the container as `eth0`, the other on the host (named `vethXXXX`) and plugs into the [[docker0 bridge]].

> [!note] How to read `eth0@if13`
> Inside the container, `5: eth0@if13` means "my interface #5 is paired with interface #13 on the other side" — i.e. the host-side end is `if13`. This is how you trace which veth on the host belongs to which container.

## Notes

### Mechanics
- Pair is always created together; you cannot have just one veth.
- Data is bidirectional: bytes pushed into end A appear on end B immediately, with no protocol in between (it's just a wire).
- Each end has its own MAC address, so the [[docker0 bridge]] can use L2 ([[OSI 7-layer model]] L2) MAC-learning to forward frames.

### Live trace example
Container side:
```
5: eth0@if17  inet 192.168.215.2/24  gateway 192.168.215.1
```
Host (Linux VM) side:
```
17: veth6ec1993@if5  master docker0
```
→ Container's interface #5 ↔ host's interface #17 are the two ends of one veth pair. The host end is enslaved to the [[docker0 bridge]] ("master docker0").

### Where it fits in the stack
veth is a virtual NIC. Cable metaphor is L1, but it carries Ethernet frames with MAC headers, so the visible behavior is L2 of the [[OSI 7-layer model]].

> [!tip] Two parts to know
> 1. veth = the cable.
> 2. [[docker0 bridge]] = the switch the cable plugs into.
> Both together get a container onto a shared layer-2 network with the host.

---

## 한국어

### 개요

veth pair는 양 끝이 뚫린 **가상 이더넷 인터페이스 한 쌍**으로, 항상 쌍으로 생기고 마법의 랜선처럼 동작한다 — 한쪽 끝에 들어간 것은 반대쪽으로 나온다. 컨테이너가 격리된 자기 [[NET namespace]]를 호스트 네트워크에 잇기 위해 사용한다. 한쪽 끝은 컨테이너 안에 `eth0`로, 다른 쪽은 호스트에 `vethXXXX`로 있고 [[docker0 bridge]]에 꽂힌다.

> [!note] `eth0@if13` 읽는 법
> 컨테이너 안에서 `5: eth0@if13`은 "내 5번 인터페이스의 반대쪽은 13번 인터페이스"라는 뜻 — 즉 호스트 쪽 끝이 `if13`. 어느 veth가 어느 컨테이너 것인지 추적할 때 이렇게 쓴다.

### 노트

#### 작동 원리
- 항상 쌍으로 생성됨. 한쪽만 만들 수 없다.
- 양방향: A 끝에 넣은 바이트가 B 끝에 즉시 나타남. 중간에 프로토콜 없음 — 그냥 선.
- 각 끝이 자기 MAC 주소를 가져서, [[docker0 bridge]]가 L2([[OSI 7-layer model]] L2) MAC 학습으로 프레임을 전달할 수 있다.

#### 라이브 추적 예시
컨테이너 쪽:
```
5: eth0@if17  inet 192.168.215.2/24  gateway 192.168.215.1
```
호스트 (Linux VM) 쪽:
```
17: veth6ec1993@if5  master docker0
```
→ 컨테이너의 5번과 호스트의 17번이 veth pair의 양 끝. 호스트 쪽 끝은 [[docker0 bridge]]에 묶여있음("master docker0").

#### 스택에서의 위치
veth는 가상 NIC. 랜선 비유는 L1이지만 MAC 헤더 달린 이더넷 프레임을 다루므로 [[OSI 7-layer model]]의 L2 동작을 보인다.

> [!tip] 두 부품 한 묶음
> 1. veth = 랜선.
> 2. [[docker0 bridge]] = 그 랜선이 꽂히는 스위치.
> 둘이 합쳐져야 컨테이너가 호스트와 공유하는 L2 네트워크에 합류한다.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
