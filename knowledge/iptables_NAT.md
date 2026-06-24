---
id: 019ea148-86d3-766b-a0a9-b4bd946f1a2c
name: iptables NAT
aliases:
  - DNAT
  - MASQUERADE
  - NAT
  - iptables
  - netfilter
updated_at: '2026-06-07T08:52:25.427Z'
summary: >-
  Linux netfilter feature that rewrites packet source or destination addresses,
  used by Docker to map host ports to containers and let containers reach the
  internet.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# iptables NAT

## Overview

iptables NAT is the Linux kernel's **address-translation engine**. For [[Docker]] it does two essential jobs: (1) **DNAT** rewrites the destination so external traffic hitting a published host port lands inside a container, and (2) **MASQUERADE** rewrites the source so outbound container traffic looks like it came from the host. Without these rules, containers would be unreachable from outside and unable to reach the internet.

> [!note] Mental model
> - **DNAT** = destination NAT = inbound address swap (host:port → container:port). Triggered by `docker run -p`.
> - **MASQUERADE** = source NAT = outbound address swap (container IP → host IP). Automatic.

## Notes

### Concrete rules Docker installs
Inbound (the literal effect of `-p 8080:80`):
```
-A DOCKER ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.215.2:80
```
"Any TCP packet arriving on any interface other than [[docker0 bridge|docker0]] with destination port 8080 — rewrite destination to `192.168.215.2:80`."

Outbound (automatic):
```
-A POSTROUTING -s 192.168.215.0/24 ! -o docker0 -j MASQUERADE
```
"Any packet from the container subnet leaving via something other than docker0 — rewrite source to the host's IP."

### Packet flow for inbound `localhost:8080 → container:80`
```
Mac localhost:8080
   ↓ iptables PREROUTING / DOCKER chain (DNAT)
   ↓ docker0 bridge
   ↓ veth pair
   ↓ container eth0
nginx :80
```

### OSI placement
iptables NAT straddles two layers of the [[OSI 7-layer model]]: it rewrites both IP address (L3) and port (L4). Port-mapping `8080:80` itself is pure L4.

### Security implication — DNAT hijacking
Root/CAP_NET_ADMIN holders can insert their own DNAT rules and redirect traffic to a malicious container. This is one form of the broader "control the route, control the traffic" attack family (see [[BGP hijacking]], [[DNS spoofing]], [[ARP spoofing]]).

```
iptables -t nat -I DOCKER 1 ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination <attacker_IP>:80
```
`-I … 1` inserts at position 1 — first match wins, so this overrides the original rule. Defense: forbid `--privileged` / `--cap-add=NET_ADMIN`, audit iptables changes (auditd, Falco), and most importantly use [[TLS]]/[[mTLS]] so a hijacker can't impersonate the real server (see [[MyEtherWallet 2018 BGP hijacking]]).

> [!warning] macOS quirk
> [[OrbStack]] / Docker Desktop publish ports via a userspace proxy (docker-proxy), so Mac → container traffic via `localhost:8080` BYPASSES iptables and goes straight to the container. DNAT hijacking experiments only work when you actually traverse iptables (e.g. hitting the VM IP directly).

---

## 한국어

### 개요

iptables NAT는 Linux 커널의 **주소 번역 엔진**이다. [[Docker]]에서는 두 가지 역할을 한다: (1) **DNAT**가 목적지를 바꿔서 호스트의 publish된 포트로 들어온 트래픽이 컨테이너 안으로 가게 하고, (2) **MASQUERADE**가 출발지를 바꿔서 컨테이너에서 나가는 트래픽이 호스트에서 온 것처럼 보이게 한다. 이 규칙들이 없으면 컨테이너는 외부에서 접근 불가, 인터넷도 못 나간다.

> [!note] 멘탈 모델
> - **DNAT** = destination NAT = 들어오는 주소 갈아끼기 (호스트:포트 → 컨테이너:포트). `docker run -p`로 트리거.
> - **MASQUERADE** = source NAT = 나가는 주소 갈아끼기 (컨테이너 IP → 호스트 IP). 자동.

### 노트

#### Docker가 설치하는 실제 규칙
들어오는 길 (`-p 8080:80`의 실체):
```
-A DOCKER ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination 192.168.215.2:80
```
"[[docker0 bridge|docker0]]가 아닌 다른 인터페이스로 들어온, 목적지 포트 8080인 TCP 패킷의 목적지를 `192.168.215.2:80`으로 갈아끼워라."

나가는 길 (자동):
```
-A POSTROUTING -s 192.168.215.0/24 ! -o docker0 -j MASQUERADE
```
"컨테이너 서브넷에서 출발해서 docker0가 아닌 곳으로 나가는 패킷의 출발지를 호스트 IP로 갈아끼워라."

#### 들어오는 트래픽 흐름 `localhost:8080 → 컨테이너:80`
```
맥 localhost:8080
   ↓ iptables PREROUTING / DOCKER 체인 (DNAT)
   ↓ docker0 bridge
   ↓ veth pair
   ↓ 컨테이너 eth0
nginx :80
```

#### OSI 위치
iptables NAT은 [[OSI 7-layer model]]의 두 계층에 걸쳐있다 — IP 주소(L3)도 포트(L4)도 갈아끼움. 포트 매핑 `8080:80` 자체는 순수 L4.

#### 보안 함의 — DNAT 하이재킹
root 권한이나 CAP_NET_ADMIN을 가진 자가 자기 DNAT 규칙을 끼워넣어 트래픽을 악성 컨테이너로 돌릴 수 있다. "경로를 쥐면 트래픽을 쥔다" 공격 가족 ([[BGP hijacking]], [[DNS spoofing]], [[ARP spoofing]] 참고)의 일종.

```
iptables -t nat -I DOCKER 1 ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination <공격자_IP>:80
```
`-I … 1`은 1번 위치에 삽입 — 첫 매칭이 이기므로 원래 규칙을 덮어쓴다. 방어: `--privileged`/`--cap-add=NET_ADMIN` 금지, iptables 변경 감사(auditd, Falco), 그리고 무엇보다 [[TLS]]/[[mTLS]]로 하이재커가 진짜 서버를 사칭 못 하게 ([[MyEtherWallet 2018 BGP hijacking]] 참고).

> [!warning] macOS 특수성
> [[OrbStack]] / Docker Desktop은 publish된 포트를 유저스페이스 프록시(docker-proxy)로 처리해서, 맥 → 컨테이너 트래픽이 `localhost:8080`을 거치면 iptables를 **우회**하고 컨테이너로 직접 꽂힌다. DNAT 하이재킹 실험은 실제로 iptables를 통과하는 경로(예: VM IP 직접)에서만 작동.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
