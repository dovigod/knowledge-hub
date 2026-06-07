---
id: 019ea148-86cf-714c-870c-21e12645b0d4
name: docker0 bridge
aliases:
  - docker bridge
  - docker0
updated_at: '2026-06-07T08:52:25.423Z'
summary: >-
  Default virtual Layer-2 switch that Docker creates on the host to connect
  container veth ends to each other and the outside.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# docker0 bridge

## Overview

`docker0` is the default **virtual switch (bridge)** that Docker creates on the host. It's a Layer-2 device (in [[OSI 7-layer model]] terms) into which the host-side end of every container's [[veth pair]] gets plugged. It also has its own IP (e.g. `192.168.215.1`) and serves as the default gateway for containers on that bridge — so it's a switch with a router hat.

> [!note] Two jobs
> 1. **Switch (L2):** forwards Ethernet frames between containers on the same bridge using MAC learning.
> 2. **Gateway (L3):** owns an IP so containers have somewhere to send packets destined for the outside world.

## Notes

### Topology
```
[container A eth0] ── veth ── [vethAAA@docker0]
                                     │
[container B eth0] ── veth ── [vethBBB@docker0]
                                     │
                                  docker0 (192.168.215.1)
                                     │
                              host routing / [[iptables NAT]]
                                     │
                              physical network / internet
```

### Why containers can reach each other automatically
Because they're all on the same L2 segment via docker0, container A can ping container B's IP without any extra config — the bridge forwards the frames.

### Why they don't reach the outside automatically (the asymmetric part)
Outbound packets can leave because the host masquerades them ([[iptables NAT]] MASQUERADE rule on docker0 traffic). Inbound packets cannot reach a container until you explicitly publish a port with `-p` (creates a DNAT rule — see [[Docker port mapping]] and [[iptables NAT]]).

> [!tip] On macOS
> `docker0` lives inside the Linux VM that [[OrbStack]] / Docker Desktop runs. You won't see it on the Mac host.

---

## 한국어

### 개요

`docker0`는 Docker가 호스트에 만드는 기본 **가상 스위치(브리지)**다. [[OSI 7-layer model]] 기준 L2 장비이며, 모든 컨테이너 [[veth pair]]의 호스트 쪽 끝이 여기에 꽂힌다. 자기 IP(예: `192.168.215.1`)도 가져서 그 브리지에 묶인 컨테이너들의 기본 게이트웨이 역할도 한다 — 라우터 모자를 쓴 스위치.

> [!note] 두 가지 역할
> 1. **스위치 (L2):** 같은 브리지의 컨테이너들 사이 이더넷 프레임을 MAC 학습으로 전달.
> 2. **게이트웨이 (L3):** IP를 가지므로 컨테이너가 외부로 나가는 패킷의 다음 목적지가 된다.

### 노트

#### 토폴로지
```
[컨테이너 A eth0] ── veth ── [vethAAA@docker0]
                                     │
[컨테이너 B eth0] ── veth ── [vethBBB@docker0]
                                     │
                                 docker0 (192.168.215.1)
                                     │
                              호스트 라우팅 / [[iptables NAT]]
                                     │
                              물리 네트워크 / 인터넷
```

#### 컨테이너끼리 자동으로 통하는 이유
모두 docker0를 통해 같은 L2 세그먼트에 있어서, A 컨테이너가 별도 설정 없이 B의 IP로 ping이 간다 — 브리지가 프레임을 전달.

#### 외부로는 자동인데 외부에서 오는 건 자동이 아닌 이유 (비대칭)
나가는 패킷은 호스트가 출발지를 자기 IP로 바꿔서(MASQUERADE, [[iptables NAT]]) 내보내므로 그냥 나간다. 들어오는 패킷은 `-p`로 포트를 명시적으로 publish해야만 컨테이너까지 도달한다 (DNAT 규칙 생성 — [[Docker port mapping]], [[iptables NAT]] 참고).

> [!tip] macOS에서는
> `docker0`는 [[OrbStack]] / Docker Desktop이 돌리는 Linux VM 안에 있다. 맥 호스트에선 안 보임.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
