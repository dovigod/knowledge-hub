---
id: 019ea148-9a6d-71cc-bc4f-2b13352af0de
name: Docker port mapping
aliases:
  - docker -p
  - docker port forwarding
  - 포트 매핑
updated_at: '2026-06-07T08:52:30.445Z'
summary: >-
  Mechanism that exposes a container's internal port to the host via `-p
  host:container`, implemented as an iptables DNAT rule.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Docker port mapping

## Overview

Docker port mapping (`-p HOST:CONTAINER`) opens a window through the container's network wall: traffic hitting the host on `HOST` port is forwarded to `CONTAINER` port inside the container. It is NOT configured in the [[Dockerfile]] — `EXPOSE` in a Dockerfile is purely documentation. The real mapping happens at run time (`docker run -p`) or in `docker-compose.yml`.

> [!note] Left:Right rule
> `-p 8080:3000` → left = **host** (your Mac) port, right = **container** internal port. `localhost:8080` on the host reaches the app on port 3000 inside the container. Mnemonic: `host:container`.

## Notes

### Why mapping is needed at all
Containers live in isolated [[NET namespace]]s, so their port table is invisible from outside. Without `-p`, the container is an island — you can reach it from another container on the same [[docker0 bridge]] by IP, but nothing on the host's external interface can. `-p` punches a hole through.

### Multi-service collisions
Services can all use the same **internal** port; only the **host** side must be unique:
```yaml
user-service:    ports: ["3000:3000"]
clutch-service:  ports: ["3010:3000"]   # same 3000 inside, different on host
```

### What it actually does (the iptables side)
`-p 8080:80` installs an [[iptables NAT|iptables DNAT]] rule like:
```
-A DOCKER ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination <containerIP>:80
```
So port mapping is not a Docker-private concept — it's just a friendly wrapper around DNAT.

> [!warning] Dockerfile EXPOSE ≠ port mapping
> `EXPOSE 3000` is documentation only — a note to readers and tools. It does NOT open the port to the host. You still need `-p` at run time.

> [!tip] macOS path differs
> On Mac, [[OrbStack]] / Docker Desktop use a userspace proxy (docker-proxy) for published ports, so `localhost:8080` bypasses iptables. Test via the Linux VM's IP if you want the real iptables path (relevant for [[iptables NAT|DNAT hijacking]] experiments).

---

## 한국어

### 개요

Docker 포트 매핑(`-p HOST:CONTAINER`)은 컨테이너 네트워크 벽에 창문을 뚫는 것이다 — 호스트의 `HOST` 포트로 들어온 트래픽이 컨테이너 안의 `CONTAINER` 포트로 전달된다. [[Dockerfile]]에 설정하는 게 **아니다** — Dockerfile의 `EXPOSE`는 순수 문서/메모일 뿐. 실제 매핑은 실행 시점(`docker run -p`)이나 `docker-compose.yml`에서 한다.

> [!note] 왼쪽:오른쪽 규칙
> `-p 8080:3000` → 왼쪽 = **호스트**(맥북) 포트, 오른쪽 = **컨테이너** 내부 포트. 호스트에서 `localhost:8080`으로 접속하면 컨테이너 안 3000번 앱에 도달. 외우는 법: `호스트:컨테이너`.

### 노트

#### 왜 매핑이 필요한가
컨테이너는 격리된 [[NET namespace]]에 살아서 포트 테이블이 외부에서 안 보인다. `-p` 없이는 컨테이너가 외딴 섬 — 같은 [[docker0 bridge]]의 다른 컨테이너에서는 IP로 접근 가능하지만 호스트의 외부 인터페이스에서는 불가. `-p`가 구멍을 뚫는다.

#### 다중 서비스 충돌
서비스마다 **내부** 포트는 같아도 되고, **호스트** 쪽만 유일하면 됨:
```yaml
user-service:    ports: ["3000:3000"]
clutch-service:  ports: ["3010:3000"]   # 안은 둘 다 3000, 호스트만 다름
```

#### 실제로 무슨 일이 (iptables 쪽)
`-p 8080:80`은 다음과 같은 [[iptables NAT|iptables DNAT]] 규칙을 깐다:
```
-A DOCKER ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination <컨테이너IP>:80
```
즉 포트 매핑은 Docker 고유 개념이 아니라 DNAT의 친절한 포장.

> [!warning] Dockerfile EXPOSE ≠ 포트 매핑
> `EXPOSE 3000`은 문서일 뿐 — 독자/도구에게 알려주는 메모. 호스트에 포트를 열어주지 **않는다**. 실행 시 `-p`가 여전히 필요.

> [!tip] macOS 경로는 다름
> 맥에서는 [[OrbStack]] / Docker Desktop이 publish된 포트를 유저스페이스 프록시(docker-proxy)로 처리해서 `localhost:8080`은 iptables를 우회. 실제 iptables 경로를 보고 싶으면 Linux VM의 IP로 테스트 ([[iptables NAT|DNAT 하이재킹]] 실험과 관련).

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
