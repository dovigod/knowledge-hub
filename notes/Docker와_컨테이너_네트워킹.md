---
id: 019ea25e-7a95-726b-8bf0-c82451f91f29
title: Docker와 컨테이너 네트워킹
topics:
  - docker
  - container
  - namespace
  - cgroups
  - networking
  - nat
  - veth
  - bridge
  - osi
  - security
  - bgp-hijacking
  - tls
tags:
  - infra
  - docker
  - networking
  - container
  - tls
  - security
sources:
  - 019ea259-e1c1-70da-ad6a-91432ad3f013
created_at: '2026-06-07T13:56:01.301Z'
updated_at: '2026-06-07T13:56:01.301Z'
---
> [!note] TL;DR
> [[Docker]]는 새 기술이 아니라 [[Linux kernel|리눅스 커널]]의 [[namespaces]]와 [[cgroups]]를 사용하기 쉽게 포장한 도구다. 컨테이너는 [[VM]]과 달리 호스트 커널을 공유해 가볍고 빠르며, [[NET namespace]] 덕분에 여러 서비스가 똑같이 3000번 포트를 써도 충돌하지 않는다. 호스트로 꺼낼 때만 `-p 8080:3000` 같이 호스트 포트를 다르게 주면 되고, 그 변환은 [[iptables]]의 [[DNAT]]로 일어난다. 이 [[NAT]] 경로를 장악하면 트래픽도 장악되며(2018년 [[MyEtherWallet]] [[BGP hijacking]] 사건), 마지막 방어선은 [[TLS]]/[[mTLS]]다. 공격자는 도메인명은 위조할 수 있어도 [[Certificate Authority|CA]] 서명은 위조하지 못하기 때문이다.

## 1. Docker의 역사

"내 컴퓨터에선 됐는데?"는 한 세대를 괴롭힌 농담이었다. 개발자 노트북, 동료 노트북, 스테이징, 프로덕션의 OS·라이브러리 버전 차이가 매번 사고를 만들었다.

| 시기 | 사건 | 의미 |
|---|---|---|
| 2000년대 | [[VMware]]·[[VirtualBox]] 등 [[VM]] 가상화 보편화 | OS 전체를 통째로 가상화. 무겁고 부팅 수십 초. |
| 2007–2008 | [[Linux kernel]]에 [[cgroups]]([[Google]] 기여)와 [[namespaces]] 점진적 추가 | 커널 차원의 자원·시야 격리 가능. 단, 사용이 어려움. |
| 2013-03 | [[Solomon Hykes]]가 PyCon에서 [[Docker]] 데모 | 위 커널 기능을 한 줄짜리 CLI로 포장. |
| 2015 | [[OCI]] 표준 + [[Kubernetes]] | 이미지/런타임 표준화, 오케스트레이션 등장. |
| 현재 | 사실상 표준 배포 단위 | "Build once, run anywhere". |

> [!tip] 핵심
> Docker는 **새 기술의 발명이 아니라 기존 커널 기능의 포장**이다. 그래서 macOS·Windows에는 "도커가 없고", 그 안에서 작은 Linux VM이 도는 것이다.

## 2. 어떻게 작동하는가 — VM vs 컨테이너

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│           VM 방식            │     │       컨테이너 방식           │
├─────────────────────────────┤     ├─────────────────────────────┤
│  App A   │   App B          │     │  App A   │   App B          │
│  Bins/Libs│  Bins/Libs      │     │  Bins/Libs│  Bins/Libs      │
│  Guest OS │  Guest OS  ←수GB │     │  ───── 호스트 커널 공유 ─────  │
│  Hypervisor                  │     │  Container Runtime (Docker)  │
│  Host OS                     │     │  Host OS                     │
│  Hardware                    │     │  Hardware                    │
└─────────────────────────────┘     └─────────────────────────────┘
   부팅: 수십 초                       부팅: 1초 이하
   크기: 수 GB                         크기: 수십~수백 MB
```

흐름은 **Dockerfile(레시피) → 이미지(불변 스냅샷) → 컨테이너(실행 인스턴스)**다. 격리는 [[namespaces]](시야)와 [[cgroups]](자원) 둘이 합쳐 만든다.

> [!example] Dockerfile 한 토막
> ```dockerfile
> FROM node:20-alpine
> WORKDIR /app
> COPY package.json .
> RUN npm install
> COPY . .
> EXPOSE 3000           # 문서 표시일 뿐, 실제 매핑 아님
> CMD ["node", "server.js"]
> ```

## 3. 로컬에서 직접 돌리는 것 대비 무슨 이점이 있나

| 항목 | 직접 설치 | Docker |
|---|---|---|
| 환경 재현 | 동료 OS/버전 다르면 깨짐 | 이미지 그대로 재현 |
| 버전 충돌 | Postgres 14/16 동시 쓰기 어려움 | 이미지 두 개 띄우면 끝 |
| 청소 | `brew uninstall` 후 잔재 남음 | `docker rm` 하면 깨끗 |
| 인프라 일괄 기동 | 각자 따로 실행 | `docker compose up`로 [[Redis]]+[[PostgreSQL]]+[[RabbitMQ]] 한 번에 |
| 프로덕션 동일성 | 다름 | 같은 이미지로 운영 |

> [!tip] 실무 패턴
> `yarn up:infra` 같은 npm/pnpm 스크립트가 내부적으로 `docker compose up -d`를 호출해 [[Redis]]·[[PostgreSQL]]·[[RabbitMQ]]를 한 번에 띄우는 패턴이 흔하다.

## 4. 포트 매핑이란 — Dockerfile에는 사실 없다

`Dockerfile`의 `EXPOSE 3000`은 "이 이미지는 3000을 쓴다"는 **문서·메타데이터일 뿐**이다. 실제 포트 매핑은 실행 시점에 일어난다.

```bash
docker run -p 8080:3000 my-app
#        호스트포트 ↑    ↑ 컨테이너포트
```

```
┌──── Host (macOS / Linux) ──────────────────┐
│   localhost:8080 ──┐                       │
│                    │ DNAT                  │
│                    ▼                       │
│  ┌── Container ─────────────────────┐      │
│  │  app.listen(3000) ← 진짜 3000   │      │
│  └─────────────────────────────────┘      │
└────────────────────────────────────────────┘
```

> [!note]
> 컨테이너는 격리된 벽이고, `-p`는 그 벽에 창문을 뚫는 일이다. 창문 위치(호스트 포트)와 안쪽 방 번호(컨테이너 포트)는 달라도 된다.

## 5. namespaces와 cgroups

[[Linux kernel]]이 컨테이너 격리에 쓰는 두 축이다. **namespaces는 "시야 격리", cgroups는 "자원 격리"**다.

### namespaces — 시야

| namespace | 격리하는 것 | 효과 |
|---|---|---|
| `pid` | 프로세스 ID 공간 | 호스트의 PID 5678이 컨테이너 안에서는 PID 1로 보임 |
| `net` | 네트워크 스택(인터페이스/IP/포트/iptables/라우팅) | 컨테이너 둘이 같은 3000을 써도 충돌 없음 |
| `mnt` | 마운트 트리 | 자기만의 파일시스템 뷰 |
| `uts` | 호스트네임/도메인 | `hostname`이 다르게 보임 |
| `user` | UID/GID 매핑 | 컨테이너의 root가 호스트의 일반 유저일 수 있음 |
| `ipc` | System V IPC / POSIX 메시지 큐 | 공유 메모리 격리 |

### cgroups — 자원

```bash
docker run --memory=512m --cpus=1.5 my-app
```

- 메모리 한도 초과 시 [[OOM killer]]가 그 컨테이너만 종료.
- CPU/디스크 IO도 동일하게 상한·가중치 설정 가능.

> [!tip]
> "namespaces로 시야를 가두고, cgroups로 자원을 가둔다" — 이 두 개가 합쳐진 게 컨테이너다. Docker는 이걸 손쉽게 다루는 손잡이다.

## 6. 각 서비스가 3000을 써도 컨테이너 내부에선 괜찮은 이유

포트를 바꿔치기 하는 게 아니라 **둘 다 진짜로 3000을 듣는다**. 단지 서로 다른 [[NET namespace]]에서 듣고 있을 뿐이다.

```
     ┌── 호스트 ────────────────────────────────┐
     │  포트 장부(호스트 namespace)              │
     │  3000: 안 씀                             │
     │                                          │
     │  ┌── Container A (NET ns #1) ──┐         │
     │  │  포트 장부                  │         │
     │  │  3000: 쓰는 중 (app A)     │         │
     │  └────────────────────────────┘         │
     │                                          │
     │  ┌── Container B (NET ns #2) ──┐         │
     │  │  포트 장부                  │         │
     │  │  3000: 쓰는 중 (app B)     │         │
     │  └────────────────────────────┘         │
     └──────────────────────────────────────────┘
```

> [!example] 비유
> 아파트 101호와 102호 둘 다 자기 집에 "1번 방"이 있어도 부딪히지 않는다. 충돌은 **공용 현관(호스트)으로 꺼낼 때만** 생긴다. 그래서 `-p 3010:3000`, `-p 3011:3000`처럼 호스트 측 번호만 다르게 준다.

## 7. NET namespace는 정확히 뭐고, 포트 할당은 어떻게 이루어지나

[[NET namespace]]는 **네트워크 스택 한 벌을 통째로 복제**한 것이다. 각각 자기만의:

- 네트워크 인터페이스(`eth0`, `lo`)
- IP 주소·라우팅 테이블
- 포트 점유 장부 (= 커널 안 `struct net`의 포트 해시 테이블)
- `iptables` 규칙
- 소켓 테이블

을 가진다.

`bind()` 시 일어나는 일:

```
app.listen(3000)
     │
     ▼
 bind(fd, 0.0.0.0:3000)  ← 시스템 콜
     │
     ▼
 커널: "이 프로세스가 속한 NET namespace의 포트 장부를 확인"
     │
     ├─ 비어 있음 → 장부에 기록, 성공
     └─ 이미 차 있음 → EADDRINUSE(이미 사용 중)
```

장부 자체가 namespace마다 별도이기 때문에, **다른 namespace의 3000은 보이지도 충돌하지도 않는다**.

바깥에서 컨테이너 안으로 들어오려면 별도의 다리가 필요하다 — 그게 [[veth]] pair, [[bridge|docker0]], `iptables` [[DNAT]] 조합이다(아래에서 자세히).

## 8. macOS에서는 어떻게 동작하나

macOS에는 [[Linux kernel]]이 없다. 그래서 **Docker Desktop이나 [[OrbStack]]은 작은 Linux VM을 띄우고, 그 안에서 진짜 컨테이너가 돈다**.

```
┌─────── macOS (darwin/arm64) ─────────────────┐
│  docker CLI ───────────┐                     │
│                        │ gRPC/HTTP            │
│  ┌── Linux VM (OrbStack/Docker Desktop) ──┐ │
│  │  Docker Daemon (linux/arm64)            │ │
│  │  ┌── Container A ── Container B ──┐    │ │
│  │  │  eth0          eth0            │    │ │
│  │  └─────── docker0 (bridge) ──────┘    │ │
│  │  iptables NAT 규칙은 *여기* 있다       │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

`docker version`을 치면 Client는 `darwin/arm64`, Server는 `linux/arm64`로 나오는 게 이 구조 때문이다. 맥 호스트 쉘에서 `iptables`를 쳐도 보이지 않는다 — VM 안에 있어서다.

> [!example] 데모 시나리오
> ```bash
> docker run -d -p 8001:80 --name a nginx
> docker run -d -p 8002:80 --name b nginx
> # 둘 다 내부적으로 80포트 LISTEN — 충돌 없음
> docker exec a ps aux       # nginx가 PID 1
> docker exec a ip addr      # eth0@if13 같은 가상 인터페이스
> # 맥에서: curl localhost:8001 → A, curl localhost:8002 → B
> ```

| 위치 | 자주 쓰는 명령 |
|---|---|
| 맥 호스트 | `docker ps`, `docker inspect`, `docker exec -it <c> sh` |
| 컨테이너 안 | `ps`, `ip addr`, `netstat -tlnp` |
| VM 안 | `orb`(OrbStack)으로 진입 → `iptables -t nat -L -n` |

## 9. veth / bridge / NAT 상세

### veth pair — "양 끝 뚫린 마법의 랜선"

`veth`는 한쪽에 들어간 패킷이 반대쪽으로 그대로 나오는 가상 이더넷 페어다. 한 끝은 컨테이너의 `eth0`, 반대 끝은 호스트의 `vethXXXXX`로 꽂힌다.

### docker0 — 가상 스위치 겸 게이트웨이

`docker0`는 호스트에 만들어지는 [[Linux bridge]]다. 모든 컨테이너의 호스트 측 `veth`가 여기 꽂혀서 서로 통신한다. 동시에 게이트웨이(IP `172.17.0.1` 같은 형태)이기도 하다.

```
       Container A                          Container B
   ┌──────────────────┐                  ┌──────────────────┐
   │ eth0 172.17.0.2  │                  │ eth0 172.17.0.3  │
   └────┬─────────────┘                  └────┬─────────────┘
        │veth pair                            │veth pair
   ┌────▼─────────────┐                  ┌────▼─────────────┐
   │ vethA (host)     │                  │ vethB (host)     │
   └────┬─────────────┘                  └────┬─────────────┘
        └────────────┬───────────────────────┘
               ┌────▼────┐
               │ docker0 │  ← bridge, 172.17.0.1
               └────┬────┘
                    │
              iptables NAT
              (DNAT / MASQUERADE)
                    │
                  eth0 (host)
                    │
                  외부 인터넷
```

### NAT 두 방향

| 방향 | 종류 | 무엇을 바꾸나 |
|---|---|---|
| 컨테이너 → 외부 | `MASQUERADE` (SNAT) | 출발지 IP를 호스트 IP로 가장 |
| 외부 → 컨테이너 (`-p`) | `DNAT` | 목적지 `호스트:8080`을 `컨테이너IP:3000`으로 갈아끼움 |

> [!example] 실측
> `docker run -d -p 8080:80 nginx` 후
> - 컨테이너 안: `5: eth0@if17`
> - 호스트(VM) 안: `17: vethXXXX@if5 master docker0` ← 양 끝이 서로를 가리킴
> - `iptables -t nat -L -n`에 `DOCKER` 체인의 `DNAT tcp -- ... dpt:8080 to:172.17.0.2:80` 한 줄
> - `curl localhost:8080` → HTTP 200

## 10. OSI 7계층에서 각 요소는 어디?

| 계층 | 컨테이너 네트워킹 요소 |
|---|---|
| L7 응용 | [[nginx]], HTTP 요청, REST API |
| L4 전송 | TCP/UDP, 포트 번호, `bind()`/`listen()` |
| L3 네트워크 | IP 주소, 라우팅, `iptables` NAT(DNAT/MASQUERADE) |
| L2 데이터링크 | `veth`, `docker0`(bridge), MAC 주소 |
| L1 물리 | `veth` 가상 랜선 (실제 케이블 대체) |

> [!note]
> `iptables` NAT은 L3 헤더의 IP와 L4 헤더의 포트를 같이 갈아끼우므로 **L3+L4 양다리**다. OSI에 들어가지 않는 것: [[PID namespace]]·[[cgroups]] 같은 **OS 격리 메커니즘**(OSI 바깥). [[NET namespace]] 자체는 L2~L4 스택 한 벌을 통째로 복제한 추상이다.

캡슐화는 보낼 때 위→아래로 봉투를 씌우고(HTTP → TCP → IP → Ethernet), 받을 때 아래→위로 봉투를 벗긴다.

## 11. DNAT 조작 — 트래픽을 가로채는 한 줄

[[iptables]]의 [[NAT|NAT 테이블]]을 쥔 자가 행선지를 정한다. `root` 권한이 필요하다.

> [!warning] 합법적 자기 환경에서만
> 아래는 자기 노트북·VM에서 동작 원리를 배우기 위한 시연이다. 남의 시스템에 시도하면 명백한 범죄.

```bash
# 컨테이너 A를 정상 띄움
docker run -d -p 8080:80 --name A nginx
docker run -d --name B httpd     # B는 -p 없음, 내부 IP만 있음

B_IP=$(docker inspect -f '{{.NetworkSettings.IPAddress}}' B)

# 8080으로 오는 트래픽을 B로 빼돌리는 한 줄
sudo iptables -t nat -I DOCKER 1 \
  ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination ${B_IP}:80
```

결과 관찰:

| 접근 경로 | 응답 | 이유 |
|---|---|---|
| 맥 호스트에서 `curl localhost:8080` | 여전히 A | OrbStack/Docker Desktop의 `docker-proxy`가 iptables를 거치지 않고 직접 포워딩 |
| VM 내부에서 `curl <VM_IP>:8080` | B | `PREROUTING → DOCKER` 체인이 정상 통과되며 DNAT 적중 |

복구:
```bash
sudo iptables -t nat -D DOCKER \
  ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination ${B_IP}:80
```

> [!tip] 교훈
> DNAT 조작은 **그 트래픽이 실제로 iptables를 통과할 때만** 가로챌 수 있다. macOS의 `localhost` 경로는 `docker-proxy`가 우회시켜서 무효였다. 원리는 같다: **경로를 쥔 자가 트래픽을 쥔다**.

## 12. 실생활 공격과 방어

### 경로 장악 공격의 다양한 얼굴

| 계층 | 공격 | 방법 |
|---|---|---|
| L7 | 악성 사이드카, [[docker.sock]] 노출 | 같은 파드/호스트에 코드 주입 |
| L3/L4 | 컨테이너 [[iptables]] 조작, 클라우드 메타데이터 하이재킹 | `--privileged` 컨테이너로 NAT 변조 |
| L2 | [[ARP spoofing]], [[DHCP]] 스푸핑, Evil Twin Wi-Fi | 같은 브로드캐스트 도메인에서 |
| 인터넷 | [[BGP hijacking]], [[DNS]] 캐시 포이즈닝 | 전 세계 라우팅 자체를 속임 |

블록체인 도메인에선 [[BGP]]/[[DNS]] 하이재킹으로 지갑·[[DEX]]의 가짜 사이트를 보여주는 공격이 반복돼 왔다(대표가 [[MyEtherWallet]] 2018).

### 3축 방어

> [!tip] 1축 — 경로 장악 자체를 막기
> - 컨테이너에 `--privileged` 금지, `--cap-drop=ALL` 후 필요한 cap만 추가
> - Kubernetes [[NetworkPolicy]] / [[Calico]] / [[Cilium]]
> - L2: [[Dynamic ARP Inspection|DAI]], [[DHCP Snooping]]
> - 인터넷 라우팅: [[RPKI]]

> [!tip] 2축 — 장악당해도 무력화 (가장 강력)
> - [[TLS]] / [[mTLS]] — 도메인이 아니라 **인증서 신원**으로 검증
> - [[Service mesh]]([[Istio]], [[Linkerd]])로 mTLS 자동·강제화
> - 모바일/지갑은 [[certificate pinning|인증서 피닝]]

> [!tip] 3축 — 탐지
> - `iptables` 규칙 감사 / [[Falco]] 같은 런타임 보안
> - eBPF 기반 관측, 흐름 비정상치 탐지

실무 베스트 프랙티스: **mTLS + NetworkPolicy + 특권 금지**.

## 13. MyEtherWallet 2018 BGP 하이재킹과 TLS 방어선

### 사건 요약 — 2018-04-24

```
공격자 AS  ──► 인터넷에 거짓 BGP 광고
              "AWS Route 53의 IP 대역(/24)을
               우리가 더 짧은 경로로 갖고 있다"
                       │
                       ▼
ISP들이 그 광고를 받아들임 (more-specific prefix가 우선)
                       │
                       ▼
myetherwallet.com DNS 조회 트래픽이 공격자 DNS로 감
                       │
                       ▼
공격자 DNS는 가짜 IP를 응답 → 사용자 브라우저가 가짜 사이트 접속
                       │
                       ▼
가짜 사이트에 개인키 입력 → 약 215 ETH 탈취
```

> [!warning] 결정적 디테일
> 가짜 사이트는 **유효한 TLS 인증서가 없었다.** 브라우저는 빨간 경고를 띄웠고, **그 경고를 무시한 사람만 털렸다.** [[TLS]]가 마지막 방어선으로 작동한 사례다.

### 왜 막혔나 — 도메인은 위조해도 [[Certificate Authority|CA]] 서명은 못 한다

[[TLS]] 인증서에는 두 가지가 있다:
1. 도메인 이름(`Common Name` / `SAN`) — **누구나 적을 수 있다**.
2. 그 위에 찍힌 **신뢰된 CA의 디지털 서명** — CA의 비공개 키가 있어야 위조 가능.

공격자는 도메인 이름은 어떻게든 적을 수 있지만, [[DigiCert]]·[[Let's Encrypt]] 같은 신뢰된 CA의 비공개 키를 갖고 있지 않다. 그래서 가짜 사이트의 인증서는 **자체 서명(self-signed)** 일 수밖에 없고, 브라우저는 이를 거부한다.

### TLS 데모 — openssl + nginx로 재현

#### 1) 인증서 3종 만들기

```bash
# (1) 루트 CA — 우리 가상의 신뢰 기관
openssl req -x509 -newkey rsa:2048 -keyout ca.key -out ca.crt \
  -days 365 -nodes -subj "/CN=MyTrustedRootCA"
```
줄별 의미:
- `req -x509`: **자체 서명** 인증서를 직접 만든다(루트는 자기 자신이 서명).
- `-newkey rsa:2048`: 새 RSA 키 쌍 생성.
- `-keyout ca.key`: 비공개 키 파일.
- `-out ca.crt`: 공개 인증서.
- `-days 365`: 1년 유효.
- `-nodes`: 키에 암호를 걸지 않음(실습 편의).
- `-subj "/CN=MyTrustedRootCA"`: 발급자 이름.

```bash
# (2) 정품 서버 인증서 — CSR을 만들고 CA가 서명
openssl req -newkey rsa:2048 -keyout bank.key -out bank.csr \
  -nodes -subj "/CN=bank.local"

openssl x509 -req -in bank.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out bank.crt -days 365
```
- 첫 명령은 **CSR(인증서 서명 요청)**. 서버가 "내 이름은 `bank.local`입니다"라고 신청서를 쓰는 것.
- 두 번째 명령은 **CA가 그 신청서에 서명**해 정식 인증서로 만들어 줌. `issuer=MyTrustedRootCA`로 박힌다.

```bash
# (3) 공격자 인증서 — CA 없이 자체 서명, issuer=bank.local로 흉내
openssl req -x509 -newkey rsa:2048 -keyout evil.key -out evil.crt \
  -days 365 -nodes -subj "/CN=bank.local"
```
- 도메인 이름은 `bank.local`로 똑같이 박을 수 있다(누구든 적을 수 있으니).
- 하지만 `MyTrustedRootCA`의 비공개 키가 없으므로 **자체 서명**일 수밖에 없다. issuer == subject == `bank.local`.

#### 2) nginx 두 서버, 겉보기 동일·인증서만 다르게

```nginx
# /etc/nginx/conf.d/bank.conf  (정품)
server {
    listen 8443 ssl;
    server_name bank.local;
    ssl_certificate     /certs/bank.crt;   # CA 서명된 인증서
    ssl_certificate_key /certs/bank.key;
    location / { return 200 "REAL bank\n"; }
}

# /etc/nginx/conf.d/evil.conf  (공격자가 운영하는 가짜)
server {
    listen 9443 ssl;
    server_name bank.local;
    ssl_certificate     /certs/evil.crt;   # 자체 서명
    ssl_certificate_key /certs/evil.key;
    location / { return 200 "STOLEN keys\n"; }
}
```
- `server_name`도, 인사말 빼고 응답도 동일. **차이는 오직 인증서**.

#### 3) curl로 세 시나리오 검증

```bash
# 시나리오 1: 정품 + CA 신뢰 → 성공
curl --resolve bank.local:8443:127.0.0.1 \
     --cacert ca.crt \
     https://bank.local:8443/
# → "REAL bank"
```
- `--resolve`: DNS를 우회해 도메인→IP 매핑을 강제. (실제 [[DNS]] 하이재킹된 상황을 흉내내는 동시에, 정품 도메인이 정품 서버를 가리키도록 한다.)
- `--cacert ca.crt`: "이 CA를 믿겠다"고 신뢰 앵커 지정.
- 도메인 일치 ✅ + CA 서명 검증 ✅ → 통과.

```bash
# 시나리오 2: 가짜 사이트로 우회됨 + CA 신뢰 → 차단! (MEW의 교훈)
curl --resolve bank.local:9443:127.0.0.1 \
     --cacert ca.crt \
     https://bank.local:9443/
# → curl: (60) SSL certificate problem: self-signed certificate
```
- DNS 우회를 당해 **공격자 서버로 갔다**. 도메인 이름은 `bank.local`로 동일.
- 하지만 인증서가 자체 서명이라 우리가 믿는 CA의 서명이 없음 → **연결 거부**.
- → BGP/DNS 하이재킹에 완전히 당해도 TLS가 마지막에 막아 준다.

```bash
# 시나리오 3: 가짜 사이트 + 검증 끔(-k) → STOLEN
curl --resolve bank.local:9443:127.0.0.1 -k \
     https://bank.local:9443/
# → "STOLEN keys"
```
- `-k`(`--insecure`)는 인증서 검증을 끈다. **브라우저 경고를 무시한 사용자**와 같은 행동.
- 이것이 MEW 2018에서 실제 자산을 잃은 경로다.

> [!warning] 핵심 정리
> - **하이재킹** = 경로를 속이는 일(BGP/DNS/ARP).
> - **TLS** = 신원을 검증하는 일(누구와 통신 중인가).
> - 경로를 완전히 빼앗겨도 TLS의 인증서 검증을 끄지 않으면 차단된다.
> - 실무에서는 [[mTLS]]·서비스 메시·인증서 피닝으로 이를 **자동·강제화**한다.

## 다이어그램

[[canvas/Docker와_컨테이너_네트워킹.canvas|개념도]]

---

## English

> [!note] TL;DR
> [[Docker]] is not a new technology — it's a friendly wrapper over the [[Linux kernel]]'s [[namespaces]] and [[cgroups]]. Unlike a [[VM]], containers share the host kernel, so they are tiny and start in under a second. Thanks to the [[NET namespace]], many services can listen on port 3000 at once without conflict; the conflict only happens when you expose them to the host, which is why `-p 8080:3000` differs only on the host side. That mapping is implemented by [[iptables]] [[DNAT]]. Whoever owns the [[NAT]] path owns the traffic — that's exactly how the 2018 [[MyEtherWallet]] [[BGP hijacking]] worked. The last line of defense is [[TLS]]/[[mTLS]]: attackers can spoof a domain name, but they cannot forge a trusted [[Certificate Authority|CA]]'s signature.

### 1. A short history of Docker

"It works on my machine" was a generation-defining joke. Slight differences between developer laptops, staging, and production produced incident after incident.

| Era | Event | Why it matters |
|---|---|---|
| 2000s | [[VMware]]/[[VirtualBox]]-style [[VM]] virtualization | Full OS per workload — heavy, slow boot |
| 2007–2008 | [[cgroups]] (contributed by [[Google]]) and [[namespaces]] land in the [[Linux kernel]] | Kernel-level isolation possible — but raw to use |
| 2013-03 | [[Solomon Hykes]] demos [[Docker]] at PyCon | Wraps the kernel features into a one-line CLI |
| 2015 | [[OCI]] standards + [[Kubernetes]] | Image/runtime standards and orchestration |
| Today | De-facto unit of deployment | "Build once, run anywhere" |

> [!tip] The key insight
> Docker did not invent new isolation tech — it **packaged existing kernel features**. That's why macOS and Windows don't really "have Docker"; they spin up a tiny Linux VM and run containers inside it.

### 2. How it works — VM vs container

```
┌─────────────────────────────┐     ┌─────────────────────────────┐
│            VM                │     │         Container            │
├─────────────────────────────┤     ├─────────────────────────────┤
│  App A   │   App B          │     │  App A   │   App B          │
│  Bins/Libs│  Bins/Libs      │     │  Bins/Libs│  Bins/Libs      │
│  Guest OS │  Guest OS  ←GBs  │     │  ── shared host kernel ──   │
│  Hypervisor                  │     │  Container Runtime (Docker)  │
│  Host OS                     │     │  Host OS                     │
│  Hardware                    │     │  Hardware                    │
└─────────────────────────────┘     └─────────────────────────────┘
   Boot: tens of seconds              Boot: < 1 second
   Size: GBs                          Size: tens–hundreds of MB
```

Pipeline: **Dockerfile (recipe) → image (immutable snapshot) → container (running instance)**. Isolation = [[namespaces]] (view) + [[cgroups]] (resources).

> [!example] A Dockerfile
> ```dockerfile
> FROM node:20-alpine
> WORKDIR /app
> COPY package.json .
> RUN npm install
> COPY . .
> EXPOSE 3000           # documentation only, not a real mapping
> CMD ["node", "server.js"]
> ```

### 3. Why use Docker vs running locally?

| Concern | Plain install | Docker |
|---|---|---|
| Reproducibility | Fragile across OS/versions | Image is identical |
| Version conflicts | Hard to run Postgres 14 and 16 side by side | Two images, done |
| Cleanup | Leftovers after `brew uninstall` | `docker rm` is clean |
| Spinning up infra | Each service by hand | `docker compose up` brings up [[Redis]]+[[PostgreSQL]]+[[RabbitMQ]] together |
| Prod parity | Different from prod | Same image as prod |

> [!tip] Common pattern
> A script like `yarn up:infra` typically wraps `docker compose up -d`, starting [[Redis]], [[PostgreSQL]], and [[RabbitMQ]] in one command.

### 4. What is port mapping?

`EXPOSE 3000` in a Dockerfile is **just documentation/metadata** — it does not publish anything. Actual mapping happens at run time.

```bash
docker run -p 8080:3000 my-app
#         host port ↑   ↑ container port
```

```
┌──── Host (macOS / Linux) ──────────────────┐
│   localhost:8080 ──┐                       │
│                    │ DNAT                  │
│                    ▼                       │
│  ┌── Container ─────────────────────┐      │
│  │  app.listen(3000) ← real 3000   │      │
│  └─────────────────────────────────┘      │
└────────────────────────────────────────────┘
```

> [!note]
> A container is an isolated wall; `-p` punches a window through it. The host side (window position) and container side (room number) need not match.

### 5. namespaces and cgroups

The two axes the [[Linux kernel]] uses for container isolation. **namespaces isolate *view*, cgroups isolate *resources*.**

#### namespaces — view

| namespace | Isolates | Effect |
|---|---|---|
| `pid` | Process IDs | Host PID 5678 looks like PID 1 inside |
| `net` | Network stack (interfaces, IPs, ports, iptables, routes) | Two containers can both use port 3000 |
| `mnt` | Mount tree | Its own filesystem view |
| `uts` | Hostname / domain | Different `hostname` |
| `user` | UID/GID mapping | Container root may map to a normal host user |
| `ipc` | SysV IPC / POSIX message queues | Shared memory isolation |

#### cgroups — resources

```bash
docker run --memory=512m --cpus=1.5 my-app
```

- Exceed the memory cap and the [[OOM killer]] terminates only that container.
- CPU and disk I/O can be capped and weighted similarly.

> [!tip]
> "namespaces cage the *view*, cgroups cage the *resources*." Combine them and you have a container. Docker is the friendly knob on top.

### 6. Why services don't fight over port 3000 inside their containers

Docker doesn't remap their ports — **both genuinely listen on 3000**. They just listen in different [[NET namespace]]s.

```
     ┌── Host ──────────────────────────────────┐
     │  Port table (host namespace)              │
     │  3000: free                              │
     │                                          │
     │  ┌── Container A (NET ns #1) ──┐         │
     │  │  Port table                 │         │
     │  │  3000: in use by app A     │         │
     │  └────────────────────────────┘         │
     │                                          │
     │  ┌── Container B (NET ns #2) ──┐         │
     │  │  Port table                 │         │
     │  │  3000: in use by app B     │         │
     │  └────────────────────────────┘         │
     └──────────────────────────────────────────┘
```

> [!example] Analogy
> Apartment 101 and 102 both have a "Room 1" — they don't collide because each is inside its own apartment. The fight only happens at the shared entrance (the host). That's why we publish them as `-p 3010:3000` and `-p 3011:3000`, where only the host-side number changes.

### 7. What exactly is a NET namespace, and how is a port allocated?

A [[NET namespace]] is **a full copy of the network stack**. Each one has its own:

- Network interfaces (`eth0`, `lo`)
- IP addresses and routing tables
- Port allocation table (the port hash inside the kernel's `struct net`)
- `iptables` rules
- Socket table

What happens on `bind()`:

```
app.listen(3000)
     │
     ▼
 bind(fd, 0.0.0.0:3000)  ← syscall
     │
     ▼
 kernel: "look up the port table of this process's NET namespace"
     │
     ├─ free → register, return success
     └─ taken → EADDRINUSE
```

Because each namespace has its own table, **other namespaces' port 3000 is invisible and cannot collide**.

To reach into a container from outside, you need a bridge — that's the [[veth]] pair, the [[bridge|docker0]] bridge, and `iptables` [[DNAT]] working together (next section).

### 8. How does it work on macOS?

macOS has no [[Linux kernel]]. So **Docker Desktop / [[OrbStack]] spins up a small Linux VM**, and the actual containers run inside that VM.

```
┌─────── macOS (darwin/arm64) ─────────────────┐
│  docker CLI ───────────┐                     │
│                        │ gRPC/HTTP            │
│  ┌── Linux VM (OrbStack/Docker Desktop) ──┐ │
│  │  Docker Daemon (linux/arm64)            │ │
│  │  ┌── Container A ── Container B ──┐    │ │
│  │  │  eth0          eth0            │    │ │
│  │  └─────── docker0 (bridge) ──────┘    │ │
│  │  iptables NAT rules live *here*        │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

`docker version` reports Client as `darwin/arm64` and Server as `linux/arm64` because of this layout. Running `iptables` from the mac shell won't find anything — those rules live inside the VM.

> [!example] Demo
> ```bash
> docker run -d -p 8001:80 --name a nginx
> docker run -d -p 8002:80 --name b nginx
> # Both listen on 80 internally — no conflict
> docker exec a ps aux       # nginx is PID 1
> docker exec a ip addr      # virtual interface like eth0@if13
> # From mac: curl localhost:8001 → A, curl localhost:8002 → B
> ```

| Where | Useful commands |
|---|---|
| Mac host | `docker ps`, `docker inspect`, `docker exec -it <c> sh` |
| Inside container | `ps`, `ip addr`, `netstat -tlnp` |
| Inside the VM | `orb` (OrbStack) to enter → `iptables -t nat -L -n` |

### 9. veth / bridge / NAT in detail

#### veth pair — "a magic ethernet cable open at both ends"

A `veth` is a virtual ethernet pair: a packet sent into one end emerges from the other. One end is the container's `eth0`; the other is plugged into the host as `vethXXXXX`.

#### docker0 — virtual switch and gateway

`docker0` is a [[Linux bridge]] on the host. Every container's host-side `veth` plugs into it, letting containers talk to each other and to the outside. It's also their gateway (e.g. `172.17.0.1`).

```
       Container A                          Container B
   ┌──────────────────┐                  ┌──────────────────┐
   │ eth0 172.17.0.2  │                  │ eth0 172.17.0.3  │
   └────┬─────────────┘                  └────┬─────────────┘
        │veth pair                            │veth pair
   ┌────▼─────────────┐                  ┌────▼─────────────┐
   │ vethA (host)     │                  │ vethB (host)     │
   └────┬─────────────┘                  └────┬─────────────┘
        └────────────┬───────────────────────┘
               ┌────▼────┐
               │ docker0 │  ← bridge, 172.17.0.1
               └────┬────┘
                    │
              iptables NAT
              (DNAT / MASQUERADE)
                    │
                  eth0 (host)
                    │
                  internet
```

#### Two NAT directions

| Direction | Type | What it rewrites |
|---|---|---|
| Container → outside | `MASQUERADE` (SNAT) | Source IP becomes the host IP |
| Outside → container (`-p`) | `DNAT` | Destination `host:8080` becomes `containerIP:3000` |

> [!example] Observed
> After `docker run -d -p 8080:80 nginx`:
> - Inside container: `5: eth0@if17`
> - Inside host/VM: `17: vethXXXX@if5 master docker0` — both ends point to each other
> - `iptables -t nat -L -n` shows in the `DOCKER` chain: `DNAT tcp -- ... dpt:8080 to:172.17.0.2:80`
> - `curl localhost:8080` → HTTP 200

### 10. Where does each piece sit on the OSI model?

| Layer | Container-networking piece |
|---|---|
| L7 Application | [[nginx]], HTTP, REST APIs |
| L4 Transport | TCP/UDP, ports, `bind()`/`listen()` |
| L3 Network | IP, routing, `iptables` NAT (DNAT/MASQUERADE) |
| L2 Data link | `veth`, `docker0` (bridge), MAC addresses |
| L1 Physical | `veth` virtual cable (replacing a real one) |

> [!note]
> `iptables` NAT rewrites L3 (IP) and L4 (port) headers together, so it straddles **L3+L4**. Outside the OSI stack: [[PID namespace]] and [[cgroups]] — they are **OS-level isolation** rather than networking. A [[NET namespace]] is a full clone of the L2–L4 stack abstracted into one unit.

Encapsulation: outgoing packets get wrappers added top → bottom (HTTP → TCP → IP → Ethernet); incoming packets have them peeled off bottom → top.

### 11. Tampering with DNAT — one line to hijack traffic

Whoever owns the [[NAT|NAT table]] in [[iptables]] decides where traffic goes. Requires `root`.

> [!warning] Self-owned environments only
> Below is a learning demo on your own laptop/VM. Doing it to someone else's system is a crime.

```bash
docker run -d -p 8080:80 --name A nginx
docker run -d --name B httpd     # no -p; only internal IP

B_IP=$(docker inspect -f '{{.NetworkSettings.IPAddress}}' B)

# One line that diverts traffic on 8080 to B instead
sudo iptables -t nat -I DOCKER 1 \
  ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination ${B_IP}:80
```

Observations:

| Access path | Response | Why |
|---|---|---|
| `curl localhost:8080` from the Mac host | Still A | Docker Desktop/OrbStack's `docker-proxy` bypasses iptables and forwards directly |
| `curl <VM_IP>:8080` from inside the VM | B | Traffic flows through `PREROUTING → DOCKER`, hitting our DNAT |

Restore:
```bash
sudo iptables -t nat -D DOCKER \
  ! -i docker0 -p tcp --dport 8080 -j DNAT --to-destination ${B_IP}:80
```

> [!tip] Lesson
> DNAT tampering only intercepts traffic **that actually traverses iptables**. The mac's `localhost` path skipped iptables thanks to `docker-proxy`, so the rule did nothing there. The principle still holds: **own the path, own the traffic.**

### 12. Real-world attacks and defenses

#### The many faces of path hijacking

| Layer | Attack | How |
|---|---|---|
| L7 | Malicious sidecar, [[docker.sock]] exposed | Inject code into the same pod/host |
| L3/L4 | Container [[iptables]] tampering, cloud metadata hijacking | `--privileged` container rewrites NAT |
| L2 | [[ARP spoofing]], [[DHCP]] spoofing, Evil Twin Wi-Fi | Same broadcast domain |
| Internet | [[BGP hijacking]], [[DNS]] cache poisoning | Lie about routes to the whole world |

In crypto, [[BGP]]/[[DNS]] hijacks repeatedly showed up as fake wallet/[[DEX]] sites — the most famous being [[MyEtherWallet]] 2018.

#### Three axes of defense

> [!tip] Axis 1 — Prevent the hijack itself
> - No `--privileged` containers; drop all caps and add back only what's needed (`--cap-drop=ALL`)
> - Kubernetes [[NetworkPolicy]] / [[Calico]] / [[Cilium]]
> - L2: [[Dynamic ARP Inspection|DAI]], [[DHCP Snooping]]
> - Internet routing: [[RPKI]]

> [!tip] Axis 2 — Neutralize even after a hijack (the strongest)
> - [[TLS]] / [[mTLS]] — verify **identity (certificate)**, not the domain alone
> - [[Service mesh]] ([[Istio]], [[Linkerd]]) automates and enforces mTLS
> - For mobile/wallets: [[certificate pinning]]

> [!tip] Axis 3 — Detection
> - Audit `iptables` rules; runtime security like [[Falco]]
> - eBPF-based observability, flow anomaly detection

Practical best practice: **mTLS + NetworkPolicy + no privileged containers.**

### 13. MyEtherWallet 2018 BGP hijack and the TLS defense line

#### What happened — 2018-04-24

```
Attacker AS  ──► broadcasts a bogus BGP advertisement to the internet
                "AWS Route 53's /24 lives on a shorter path through us"
                          │
                          ▼
ISPs accept it (more-specific prefix wins)
                          │
                          ▼
DNS lookups for myetherwallet.com get routed to the attacker's DNS
                          │
                          ▼
Attacker DNS answers with a fake IP → user lands on a fake site
                          │
                          ▼
User enters their private key → ~215 ETH stolen
```

> [!warning] The critical detail
> The fake site **did not have a valid TLS certificate.** Browsers showed a red warning, and **only users who clicked through were robbed.** [[TLS]] was the last line of defense, and it worked for everyone who heeded it.

#### Why TLS held — anyone can spoof a domain, nobody can forge a CA signature

A [[TLS]] cert carries two things:
1. The domain name (`Common Name` / `SAN`) — **anyone can write whatever they want there.**
2. A digital signature from a trusted [[Certificate Authority|CA]] over that name — **forgeable only with the CA's private key.**

An attacker can write the right domain into a cert, but they don't possess [[DigiCert]]'s or [[Let's Encrypt]]'s private keys. So their cert is necessarily **self-signed**, and the browser refuses it.

#### TLS demo — openssl + nginx

##### 1) Build three certificates

```bash
# (1) Root CA — our pretend trust anchor
openssl req -x509 -newkey rsa:2048 -keyout ca.key -out ca.crt \
  -days 365 -nodes -subj "/CN=MyTrustedRootCA"
```
Line by line:
- `req -x509`: produce a self-signed certificate directly (a root signs itself).
- `-newkey rsa:2048`: generate a fresh RSA key pair.
- `-keyout ca.key`: write the private key to this file.
- `-out ca.crt`: write the public certificate here.
- `-days 365`: valid for a year.
- `-nodes`: no passphrase on the key (for convenience).
- `-subj "/CN=MyTrustedRootCA"`: the issuer's name.

```bash
# (2) Legitimate server cert — CSR + CA-signed
openssl req -newkey rsa:2048 -keyout bank.key -out bank.csr \
  -nodes -subj "/CN=bank.local"

openssl x509 -req -in bank.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out bank.crt -days 365
```
- First command: a **CSR (certificate signing request)** — the server saying "my name is bank.local."
- Second command: **the CA signs that request** into a real certificate. `issuer=MyTrustedRootCA` is baked in.

```bash
# (3) Attacker cert — self-signed, but with the same CN
openssl req -x509 -newkey rsa:2048 -keyout evil.key -out evil.crt \
  -days 365 -nodes -subj "/CN=bank.local"
```
- The domain name `bank.local` is identical — anyone can claim any name.
- But without the CA's private key, this can only be self-signed: issuer == subject == `bank.local`.

##### 2) Two nginx servers, indistinguishable except for certs

```nginx
# /etc/nginx/conf.d/bank.conf  (legit)
server {
    listen 8443 ssl;
    server_name bank.local;
    ssl_certificate     /certs/bank.crt;   # CA-signed
    ssl_certificate_key /certs/bank.key;
    location / { return 200 "REAL bank\n"; }
}

# /etc/nginx/conf.d/evil.conf  (attacker)
server {
    listen 9443 ssl;
    server_name bank.local;
    ssl_certificate     /certs/evil.crt;   # self-signed
    ssl_certificate_key /certs/evil.key;
    location / { return 200 "STOLEN keys\n"; }
}
```
- Same `server_name`, same plaintext payload save for the body. **The only real difference is the cert.**

##### 3) Three scenarios with curl

```bash
# Scenario 1: legit server + trust CA → success
curl --resolve bank.local:8443:127.0.0.1 \
     --cacert ca.crt \
     https://bank.local:8443/
# → "REAL bank"
```
- `--resolve`: forces a domain→IP mapping, bypassing real DNS (this simulates the **DNS hijack** and also sends us at the legit server).
- `--cacert ca.crt`: trust this CA as an anchor.
- Domain matches ✅ + CA-signed ✅ → passes.

```bash
# Scenario 2: hijacked to attacker + trust CA → blocked (the MEW lesson)
curl --resolve bank.local:9443:127.0.0.1 \
     --cacert ca.crt \
     https://bank.local:9443/
# → curl: (60) SSL certificate problem: self-signed certificate
```
- DNS hijack drops us at the attacker. Domain name still reads `bank.local`.
- But the cert is self-signed, no signature from a CA we trust → **connection refused.**
- → Even with a total BGP/DNS hijack, TLS catches us at the very last step.

```bash
# Scenario 3: attacker + disabled verification (-k) → STOLEN
curl --resolve bank.local:9443:127.0.0.1 -k \
     https://bank.local:9443/
# → "STOLEN keys"
```
- `-k` (`--insecure`) disables verification — equivalent to **a user clicking past the browser warning**.
- That is exactly how MEW 2018 victims lost their ETH.

> [!warning] Takeaway
> - **Hijacking** = lying about the path (BGP/DNS/ARP).
> - **TLS** = verifying the identity at the other end.
> - Even a full path-takeover is defeated by TLS as long as cert validation isn't turned off.
> - In production we make this **automatic and mandatory** via [[mTLS]], a [[service mesh]], and [[certificate pinning]].

## Diagram

[[canvas/Docker와_컨테이너_네트워킹.canvas|Concept map]]
