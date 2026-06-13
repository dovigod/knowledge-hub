---
id: 019ec06d-2dd9-77ce-a444-45c04299e9a6
title: Docker 컨테이너와 네트워킹 완전 이해
topics:
  - docker
  - container
  - networking
  - docker-compose
  - iptables
  - nat
  - bridge
  - veth
tags:
  - docker
  - docker-compose
  - networking
  - iptables
  - conntrack
  - nginx
  - host.docker.internal
  - bridge
  - veth
  - DNS
  - NAT
  - port-publishing
  - macOS
  - learning
sources:
  - 019ec069-2ae8-7578-a16a-d3e2969da7e7
created_at: '2026-06-13T10:00:41.177Z'
updated_at: '2026-06-13T10:00:41.177Z'
---
> [!summary] TL;DR
> 컨테이너는 격리된 작은 컴퓨터 — host와 단절돼 있고 그 벽을 뚫는 세 장치가 **port / volume / network**. `docker-compose`는 한 파일의 서비스를 자동으로 같은 user-defined bridge에 묶고, 내장 DNS(`127.0.0.11`)로 서비스 이름을 IP로 푼다. 내부 구조는 **network namespace + veth pair + bridge + iptables + embedded DNS**. 외부 트래픽은 [[port publishing]] → host 포트 bind → (맥은 Docker Desktop의 LinuxKit VM 경유) → VM iptables DNAT → veth/bridge → 컨테이너 도착 후 nginx가 **L7 라우팅**. `host.docker.internal:host-gateway`는 컨테이너에서 "내 맥(혹은 리눅스 host)"을 가리키게 해 주는 특수 매핑이라, 이 repo처럼 upstream을 `host.docker.internal:PORT`로 잡으면 컨테이너든 `yarn dev` 로컬 프로세스든 동일하게 도달한다.

## 1. Phase 1 — image · container · Dockerfile · port · volume · network

### 큰 그림

[[Docker container]]는 "격리된 작은 컴퓨터"다. 자기만의 파일시스템·네트워크·프로세스 트리를 갖고 있고, 기본적으로 host(맥북)와 완전히 단절돼 있다. 그 벽에 의도적으로 구멍을 뚫는 세 장치가 있다.

| 장치 | 잇는 대상 | 비유 |
|---|---|---|
| **port** | 컨테이너 ↔ host | 바깥에서 컨테이너로 들어가는 **출입구** |
| **volume** | 컨테이너 ↔ 디스크 | 컨테이너가 죽어도 살아남는 **창고** |
| **network** | 컨테이너 ↔ 컨테이너 | 컨테이너끼리 부르는 **내부 전화망** |

### image vs container

- **image**: 빵 굽는 **틀**. 읽기 전용 스냅샷(레이어드 파일시스템 + 메타데이터). `Dockerfile`로 빌드.
- **container**: 그 틀로 구워낸 **빵**. image 위에 쓰기 가능한 얇은 레이어를 얹어 실행한 인스턴스.
- 이미지 하나로 컨테이너 N개를 띄울 수 있고, 컨테이너 안에서 만든 변경은 컨테이너를 지우면 사라진다(그래서 [[volume]]이 필요).

### Dockerfile

```dockerfile
FROM node:20.10-alpine          # 베이스 이미지
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install
COPY . .
EXPOSE 3000                     # ⚠️ 출입구를 여는 게 아니라 '메타데이터'일 뿐
CMD ["node", "dist/server.js"]
```

> [!warning] `EXPOSE`는 포트를 열지 않는다
> `EXPOSE 3000`은 "이 이미지는 3000을 듣게 설계됐다"는 **문서/힌트**일 뿐. 실제 host로 노출하는 출입구는 `docker run -p` 또는 `docker-compose`의 `ports:`다.

### Port — 컨테이너 ↔ 외부

```yaml
services:
  postgres:
    ports:
      - "5431:5432"   # HOST_PORT:CONTAINER_PORT
  redis:
    ports:
      - "6378:6379"
  user-service:
    ports:
      - "3000:3000"
```

- **콜론 왼쪽 = host(맥)에서 접속할 번호** — 마음대로 바꿀 수 있다. 로컬에 이미 postgres가 돌고 있어 5432가 막혀 있다면 5431로 회피.
- **콜론 오른쪽 = 컨테이너 내부에서 실제로 듣는 번호** — 이미지가 정한 거라 못 바꾼다.
- 즉 **누가 부르느냐**에 따라 주소가 달라진다:

| 호출자 | postgres에 접속하는 주소 |
|---|---|
| 맥에서 DBeaver/psql | `localhost:5431` (host port 매핑) |
| 같은 compose의 컨테이너 | `postgres:5432` (network 이름 + 컨테이너 포트) |

### Volume — 컨테이너 ↔ 디스크

컨테이너 파일시스템은 휘발성이다. 살아남아야 하는 건 volume에 둬야 한다.

```yaml
services:
  postgres:
    volumes:
      - postgres:/var/lib/postgresql/data            # (a) named volume
      - ./db-dumps:/dumps                            # (b) bind mount
      - ./docker/nginx.conf:/etc/nginx/nginx.conf:ro # bind mount(파일 단위)

volumes:
  postgres:    # 위 named volume 이름 등록
```

> [!tip] named volume vs bind mount 구분법
> **콜론 왼쪽이 `이름`** 이면 → docker가 관리하는 **named volume** (`/var/lib/docker/volumes/...` 아래). 컨테이너 재생성·이미지 교체에도 데이터 유지.
> **콜론 왼쪽이 `./` 또는 `/`** 로 시작하는 경로면 → **bind mount**, host 실제 파일/디렉토리를 컨테이너에 그대로 꽂는다. 설정파일 주입·소스코드 hot-reload에 유용.

### Network — 컨테이너 ↔ 컨테이너

`docker-compose.yaml` 하나에 정의된 서비스들은 별도 설정 없이도 **같은 user-defined bridge network**에 자동으로 들어간다. 그리고 **서비스 이름이 곧 DNS 호스트명**이 된다.

```yaml
services:
  user-service:
    environment:
      DATABASE_URL: postgres://app:pw@postgres:5432/app    # ← 'postgres'가 DNS
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres                                            # 시작 순서만, 통신과 무관
    extra_hosts:
      - "host.docker.internal:host-gateway"                 # 컨테이너→맥 호스트
```

> [!note] `depends_on`은 통신과 무관
> `depends_on`은 **시작 순서**만 정하지, "준비됐는지"는 확인하지 않는다. 실제 readiness는 `healthcheck`나 앱 레벨 재시도 로직으로.

### 클라우드 매핑

| 로컬 docker-compose | 클라우드 등가 |
|---|---|
| `ports:` | 로드밸런서 / Ingress |
| `volumes:` | EBS / EFS / PersistentVolume |
| 자동 network + 서비스 이름 DNS | Service Discovery / 내부 DNS (ECS Service Connect, K8s Service) |

## 2. docker-compose 내부 네트워킹의 정체

큰 그림: **network namespace + veth pair + bridge + iptables + embedded DNS** 다섯 조각이 합쳐 만든 작품이다.

```
┌────────────────────────  Host (또는 LinuxKit VM)  ─────────────────────────┐
│                                                                              │
│   ┌─────────────┐        br-xxxxxx (172.18.0.1/16)         ┌─────────────┐  │
│   │ veth-A      │◄──── bridge (L2 software switch) ────►   │ veth-B      │  │
│   └──┬──────────┘                                          └────────┬────┘  │
│      │                                                              │       │
│      │ veth pair (가상 랜선)                          veth pair    │       │
│      ▼                                                              ▼       │
│  ┌──────────────────────┐                          ┌────────────────────┐   │
│  │ ns: user-service     │                          │ ns: postgres       │   │
│  │ eth0  172.18.0.2     │                          │ eth0  172.18.0.3   │   │
│  │ resolv.conf:         │                          │ embedded DNS       │   │
│  │   nameserver         │                          │   127.0.0.11       │   │
│  │   127.0.0.11         │                          └────────────────────┘   │
│  └──────────────────────┘                                                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

### (1) network namespace = 격리의 정체

리눅스 커널 기능. 각 namespace는 자기만의 **인터페이스 / 라우팅 테이블 / iptables / `/etc/hosts` / `/etc/resolv.conf`** 를 갖는다. 컨테이너 1개당 namespace 1개.

### (2) veth pair = namespace끼리 잇는 가상 랜선

양 끝이 한 쌍인 가상 인터페이스. 한쪽에 들어간 패킷은 반드시 반대쪽으로 나온다. 한쪽 끝은 컨테이너의 `eth0`, 다른 끝은 host의 `vethXXXXXX`.

### (3) bridge = veth들의 host쪽 끝을 묶는 L2 스위치

`docker-compose`는 프로젝트당 `<프로젝트명>_default`라는 bridge를 자동 생성(`docker network ls`에서 `br-xxxxx`). 서브넷(예: `172.18.0.0/16`)이 할당되고, 각 컨테이너에 IP가 부여된다. **bridge 자체도 IP(`172.18.0.1`)를 갖고, 이게 컨테이너들의 default gateway**.

```bash
docker network inspect <project>_default
```

### (4) embedded DNS `127.0.0.11`

각 컨테이너 namespace 내부에서만 보이는 내장 DNS 서버. `cat /etc/resolv.conf` 하면 `nameserver 127.0.0.11`이 박혀 있다.
- `postgres`를 resolve → Docker가 `172.18.0.3` 응답
- 모르는 이름은 host의 DNS로 forwarding

> [!warning] DNS는 **user-defined network에서만** 동작
> bridge에는 두 종류가 있다.
> - **default bridge** (`docker0`): `docker run`에서 `--network`를 안 주면 자동 진입. IP로는 통신되지만 **이름(DNS)으로는 못 찾는다**. 옛날에는 `--link`가 필요했던 이유.
> - **user-defined network**: `docker network create`나 `docker-compose`가 자동 생성. 내장 DNS가 이름↔IP 매핑을 해 준다.
>
> `docker-compose`는 **항상 user-defined**라 항상 DNS가 동작한다. 단독 `docker run`은 안 된다.

### (5) 외부 인터넷 나갈 때 = iptables MASQUERADE (SNAT)

컨테이너 출발지 IP는 사설(`172.18.0.2`)이라 그대로 외부로 못 나간다. host의 `nat POSTROUTING` 체인에 걸린 `MASQUERADE` 규칙이 출발지 IP를 host의 공인 IP로 바꿔서 보낸다. 응답이 돌아올 땐 [[conntrack]]이 원래 컨테이너로 되돌린다.

> [!example] conntrack은 map처럼 기록한다
> 네, 정확히 맞다. **커널 메모리의 해시 테이블**.
> 처음 패킷이 나갈 때 5-tuple(`프로토콜, 출발지IP, 출발지포트, 목적지IP, 목적지포트`)을 키로 엔트리를 만든다. NAT가 끼면 "원래 값 → 바뀐 값" 변환 정보와 **예상되는 응답의 역방향 tuple**까지 미리 기록.
> 응답 패킷이 도착하면 역방향 tuple로 조회해서 자동으로 역변환.
>
> ```bash
> sudo conntrack -L                    # 또는
> cat /proc/net/nf_conntrack
> ```

## 3. extra_hosts `host.docker.internal:host-gateway` 정밀 해부

### 무엇을 하는 한 줄인가

`extra_hosts`는 **컨테이너의 `/etc/hosts`에 한 줄 추가**한다.

```
host.docker.internal  <어떤_IP>
```

`host-gateway`는 매직 키워드라 Docker가 런타임에 "컨테이너에서 본 host의 IP"로 치환한다. 보통 bridge의 gateway IP(`172.18.0.1`).

> [!tip] 왜 명시적으로 적어 줘야 하나
> - **Docker Desktop (맥/윈)**: `host.docker.internal`이 **기본 제공**된다.
> - **Linux 네이티브 Docker**: 기본 제공되지 않는다. 그래서 `extra_hosts: ["host.docker.internal:host-gateway"]`를 적어 줘야 어디서 띄우든 같은 코드가 동작한다 — **이식성 보장용**.

### 맥의 특수성 — host.docker.internal은 VM이 아니라 **진짜 맥**이다

> [!warning] 핵심 정정
> Docker Desktop for Mac은 두 층 구조다.
>   1. **진짜 macOS 호스트** — `yarn dev`로 띄운 Node 프로세스가 맥의 `:3000`을 점유.
>   2. 그 안의 **LinuxKit VM** — 컨테이너들이 도는 곳, bridge/veth/iptables/내장 DNS는 전부 VM 안의 일.
>
> **`host.docker.internal`은 VM이 아니라 진짜 맥 호스트를 가리키도록 Docker Desktop이 일부러 배선한 특수 이름이다.** 기능의 존재 이유 자체가 "컨테이너에서 내 맥 위에서 도는 서비스(IDE의 dev server 등)에 닿게 해 줘"이기 때문.

### 외부 트래픽이 어떻게 들어와서 라우팅되는가

이 monorepo의 `nginx.conf`는 의도된 선택을 했다 — **모든 upstream을 `host.docker.internal:PORT`로 잡았다**.

```nginx
upstream user-service     { server host.docker.internal:3000; }
upstream activity-service { server host.docker.internal:3001; }
upstream clutch-service   { server host.docker.internal:3002; }

server {
  listen 80;
  location /activity      { proxy_pass http://activity-service/; }
  location ~ ^/v1/fee     { proxy_pass http://clutch-service/;   }
  location /              { proxy_pass http://user-service/;     }
}
```

> [!note] `upstream user-service`의 이름은 docker DNS가 아니다
> `nginx.conf`에 등장하는 `user-service`는 nginx의 **upstream 블록 라벨**일 뿐, docker 내장 DNS와 무관하다. `proxy_pass http://user-service/`는 그 블록의 내용(`server host.docker.internal:3000`)으로 펼쳐진다. 즉 nginx는 **컨테이너 DNS를 아예 안 쓴다**.

### 컨테이너든 `yarn dev`든 똑같이 도달하는 이유

> [!example] 분기는 nginx가 아니라 "host:3000을 누가 점유 중이냐"에서 일어난다
>
> | 시나리오 | host(맥)의 `:3000` 점유자 |
> |---|---|
> | (1) user-service를 컨테이너로 띄움 | Docker Desktop이 `ports:3000:3000`로 bind → 트래픽이 다시 VM 안 컨테이너로 |
> | (2) `yarn dev`로 로컬에서 띄움 | 맥의 Node 프로세스가 직접 bind |
>
> 두 경우 모두 **"맥의 `:3000`까지 도달"** 은 동일. nginx는 그 너머를 구분하지 않는다. 이 repo가 `server user-service:3000` (docker DNS)을 쓰지 않은 이유가 바로 이것 — "각 서비스를 로컬에서 그냥 띄워도 nginx 경유 접속이 되게" 하기 위함.

## 4. 외부 → nginx → 서비스: 정확한 트레이스

> [!warning] "타깃이 3001"이 모호하다
> 진입은 두 가지다.
> - **경우 A — nginx 경유**: 외부는 `http://host/activity`로 **포트 80**에 들어온다. `3001`은 nginx가 URL 경로 보고 내부에서 고르는 값.
> - **경우 B — 3001로 직접**: activity-service도 `ports:3001:3001`로 published돼 있으면 nginx 아예 안 거치고 곧장 컨테이너로.
>
> nginx 이야기를 하려면 **입구는 포트 80**.

### 경우 A — 외부 클라이언트가 `/activity`로 들어왔을 때

```
1. 외부 클라이언트 ─► macOS:80
                     │  맥 커널: :80을 bind()한 프로세스 찾기
                     │  → Docker Desktop (com.docker.backend)
                     ▼
2. Docker Desktop (gVisor/vpnkit) ─► VM 경계 너머 LinuxKit VM 안으로
                     │
                     ▼
3. VM의 iptables nat PREROUTING DNAT 규칙
   목적지를 VM:80 → nginx 컨테이너 IP(172.18.0.5):80 으로 재작성
                     │  (여기까지 L3/L4 라우팅 = 커널)
                     ▼
4. VM 커널 라우팅: 그 컨테이너 IP는 bridge 너머
   → bridge(br-xxxx) ─► veth pair ─► nginx eth0
                     │  (L2 전송)
                     ▼
5. nginx 프로세스가 :80 수신, HTTP 파싱
   location /activity 매칭 → upstream activity-service
   = http://host.docker.internal:3001                ← 여기서 L7 라우팅 시작
                     │
                     ▼
6. nginx → VM 밖으로 새 연결 → Docker Desktop이 다시 맥 호스트로 포워딩
                     │
                     ▼
7. 맥의 :3001
   (a) activity-service가 컨테이너면: ports:3001:3001로 published
       → Desktop이 또다시 VM 안 컨테이너로 헤어핀(hairpin)
   (b) yarn dev면: 맥의 Node 프로세스가 직접 받음
                     │
                     ▼
8. 응답: conntrack이 단계마다 기록한 역방향 매핑으로 자동 역변환되며 되감김
```

> [!warning] 흔한 오해 두 가지 채점
> - "nginx가 iptables를 분석해서 3001로 보낸다" → **❌**. iptables는 nginx 도착까지의 L3/L4 일. nginx 도착 후엔 **HTTP location / proxy_pass = L7 라우팅**.
> - "같은 bridge에 activity 컨테이너가 있으니 veth/bridge로 옆 컨테이너 직행" → **❌**. upstream이 `host.docker.internal:3001`이라 **맥 호스트까지 빠져나갔다가 published 포트로 되돌아오는 헤어핀**을 탄다. `server activity-service:3001`로 썼다면 bridge 직행했을 것.

### 경우 B — 외부에서 곧장 `:3001`로

`ports:3001:3001`로 published돼 있다면 nginx 안 거치고:

```
외부 ─► macOS:3001 ─► Docker Desktop ─► VM iptables DNAT
                                       → activity 컨테이너:3001
```

## 5. "맥은 iptables 없다며, 그럼 docker인 줄 어떻게 아냐?"

> [!example] 답: 맥 커널은 docker를 "아는" 게 아니라, **docker가 그 포트를 미리 점유하고 있어서** 도달할 뿐
>
> `ports:80:80`으로 컨테이너를 띄우면, Docker Desktop의 백그라운드 프로세스(`com.docker.backend` 등)가 **맥의 80번 포트에 평범하게 `bind()` 해서 listen 소켓을 연다.** 다른 서버 프로그램이 포트 점유하는 것과 동일하다.
>
> 외부 패킷이 도착하면 맥 커널은:
> 1. 목적지 포트(`80`)를 본다
> 2. 누가 그 포트에 bind했는지 소켓 테이블 조회 → Docker Desktop
> 3. 그 프로세스에 전달
>
> ```bash
> sudo lsof -nP -iTCP:80 -sTCP:LISTEN
> # COMMAND            PID USER  ... NODE NAME
> # com.docker.backend ... ...   ... TCP  *:80 (LISTEN)
> ```
>
> Linux 네이티브 = **커널 iptables DNAT**.
> Docker Desktop 맥 = **유저스페이스 프로세스가 포트 bind**.
>
> published 안 한 포트면 bind한 프로세스가 없으니 `connection refused`.

## 6. veth/bridge와 iptables와 nginx — "라우팅"이 두 레이어다

> [!tip] "라우팅"이 두 다른 의미로 쓰인다
>
> | 레이어 | 누가 한다 | 무엇으로 판단 | 의미 |
> |---|---|---|---|
> | **L2 전송** | veth + bridge (커널) | MAC, 물리적 경로 | "패킷이 오갈 통로" |
> | **L3/L4 라우팅** | iptables / 라우팅 테이블 (커널) | 목적지 IP·포트 | "IP·포트 보고 어디로 보낼까" |
> | **L7 라우팅** | nginx 프로세스 | URL 경로, Host 헤더, … | "HTTP 내용 보고 어느 백엔드에" |

즉 **L7에서 라우팅하는 게 아니라**, L3/L4 라우팅(커널)으로 nginx까지 패킷을 데려다 놓은 뒤 거기서 L7 라우팅(nginx)이 시작되는 **2단 구조**.

VM 진입 시퀀스 재정리:

1. 패킷이 VM 도착 (목적지 = VM 인터페이스:80)
2. **iptables nat PREROUTING DNAT** → 목적지를 nginx 컨테이너 IP:80으로 재작성 (L3/L4)
3. VM 커널 라우팅: 그 컨테이너 IP는 bridge 너머 → **bridge → veth → nginx eth0** (L2 전송)
4. nginx가 :80 수신 → HTTP 파싱 → `location /activity` 매칭 → `proxy_pass` (**L7 시작**)

질문에서 적은 "+ VM 오면 iptables가 nginx로 보내고 거기서 L7 시작"이 정확히 이 흐름이다. 한 가지만 다듬자면, "veth가 bridge에 연결돼서 네트워킹이 가능"은 **L2 전송 통로**의 의미고, **L3 라우팅 결정은 iptables/라우팅 테이블이 한다.** 통로(L2)와 결정(L3)을 분리해서 머릿속에 두면 깔끔하다.

## 7. 정리 — 한 장으로

```
┌────────────────────────────────────────────────────────────────────┐
│                       외부 클라이언트                              │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ TCP :80
                               ▼
       ┌─────────────── macOS host ──────────────────┐
       │  :80을 bind한 com.docker.backend 가 받음     │   ← iptables 아님
       │  (lsof로 확인 가능)                           │
       └────────────────────┬────────────────────────┘
                            │ gVisor/vpnkit forward
                            ▼
       ┌─────────── LinuxKit VM ─────────────────────┐
       │  iptables PREROUTING DNAT                   │   ← L3/L4
       │  VM:80 → nginx_ctn:80                       │
       │           │                                  │
       │           ▼ via bridge + veth                │   ← L2
       │     ┌─────────────────┐                      │
       │     │ nginx container │ ── HTTP parse        │
       │     │   location /act │ ── L7 routing  ──────┼─► host.docker.internal:3001
       │     └─────────────────┘                      │
       │           ▲                                  │
       │           │ 응답: conntrack 역변환            │
       └───────────┴──────────────────────────────────┘
                            ▲
                            │ 헤어핀 ── Desktop이 다시 VM 안으로
                            │           또는 yarn dev가 맥 :3001 직접 수신
                            ▼
                  activity-service container : 3001
                  또는 macOS Node process    : 3001
```

## 다이어그램

[[canvas/Docker_컨테이너와_네트워킹_완전_이해.canvas|개념도]]

---

## English

### TL;DR

A [[Docker container]] is a tiny isolated computer; **port / volume / network** are the three deliberate holes punched through the wall to the host. `docker-compose` puts every service in one file into a single **user-defined bridge** and resolves service names via an embedded DNS at `127.0.0.11`. Under the hood it's **network namespaces + veth pairs + a bridge + iptables + embedded DNS**. External traffic enters via [[port publishing]] → a process bound on the host port → (on macOS, through Docker Desktop's LinuxKit VM) → iptables DNAT → veth/bridge → the container, after which nginx does **L7 routing**. `host.docker.internal:host-gateway` is the magic mapping that lets a container reach "my actual Mac (or Linux host)"; this repo wires every upstream to `host.docker.internal:PORT` so an nginx upstream resolves the same way whether the backend is a container *or* a local `yarn dev`.

### 1. Phase 1 — image · container · Dockerfile · port · volume · network

#### Big picture

A container is an isolated little computer with its own filesystem, network, and process tree. By default it is **completely cut off from the host**. Three mechanisms deliberately bridge that wall:

| Mechanism | Connects | Mental model |
|---|---|---|
| **port** | container ↔ host | front door into the container |
| **volume** | container ↔ disk | warehouse that survives the container |
| **network** | container ↔ container | internal phone system |

#### image vs container

- **image**: the read-only mold (layered FS + metadata). Built from a `Dockerfile`.
- **container**: a running instance — image + a thin writable layer on top.
- One image → N containers. Changes inside a container vanish when it's removed (hence [[volume]]s).

#### Dockerfile

```dockerfile
FROM node:20.10-alpine
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install
COPY . .
EXPOSE 3000        # ⚠️ documentation only — does NOT open a port
CMD ["node", "dist/server.js"]
```

> [!warning] `EXPOSE` does not open a port
> It's just a hint that "this image listens on 3000". The actual host-facing door is `docker run -p` or compose `ports:`.

#### Port — container ↔ outside

```yaml
services:
  postgres:
    ports:
      - "5431:5432"   # HOST_PORT:CONTAINER_PORT
  redis:
    ports:
      - "6378:6379"
  user-service:
    ports:
      - "3000:3000"
```

- **Left of the colon = port you dial from the host** — you can change it (e.g. avoid clashing with a local Postgres on 5432).
- **Right of the colon = the port the container actually listens on** — fixed by the image.
- The address depends on **who is calling**:

| Caller | Address to reach postgres |
|---|---|
| Mac with DBeaver/psql | `localhost:5431` (host port mapping) |
| Sibling container in the same compose | `postgres:5432` (network name + container port) |

#### Volume — container ↔ disk

```yaml
services:
  postgres:
    volumes:
      - postgres:/var/lib/postgresql/data            # (a) named volume
      - ./db-dumps:/dumps                            # (b) bind mount
      - ./docker/nginx.conf:/etc/nginx/nginx.conf:ro # bind mount (single file)

volumes:
  postgres:    # register the named volume
```

> [!tip] How to tell them apart
> **Left side is a name** → **named volume** managed by Docker under `/var/lib/docker/volumes/...`. Survives container recreation and image upgrades.
> **Left side starts with `./` or `/`** → **bind mount**, jacking a real host path straight into the container. Great for injecting configs or hot-reloading source.

#### Network — container ↔ container

All services in one `docker-compose.yaml` join the **same user-defined bridge** automatically, and **the service name is its DNS host name**.

```yaml
services:
  user-service:
    environment:
      DATABASE_URL: postgres://app:pw@postgres:5432/app    # 'postgres' resolves via DNS
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres                                            # ordering only — no comms guarantee
    extra_hosts:
      - "host.docker.internal:host-gateway"                 # container → host machine
```

> [!note] `depends_on` is not a readiness probe
> It only controls **start order**, not whether the dependency is actually ready. Use `healthcheck` or app-level retries for readiness.

#### Cloud equivalents

| Local docker-compose | Cloud counterpart |
|---|---|
| `ports:` | Load balancer / Ingress |
| `volumes:` | EBS / EFS / PersistentVolume |
| Auto network + name-based DNS | Service Discovery / internal DNS (ECS Service Connect, K8s Service) |

### 2. What docker-compose networking really is

It's a composition of **network namespaces + veth pairs + a bridge + iptables + an embedded DNS** — five Linux primitives stitched together.

```
┌────────────────────────  Host (or LinuxKit VM)  ──────────────────────────┐
│                                                                            │
│   ┌─────────────┐        br-xxxxxx (172.18.0.1/16)        ┌─────────────┐  │
│   │ veth-A      │◄──── bridge (L2 software switch) ────►  │ veth-B      │  │
│   └──┬──────────┘                                         └────────┬────┘  │
│      │                                                             │       │
│      │ veth pair                                       veth pair  │       │
│      ▼                                                             ▼       │
│  ┌──────────────────────┐                          ┌────────────────────┐  │
│  │ ns: user-service     │                          │ ns: postgres       │  │
│  │ eth0  172.18.0.2     │                          │ eth0  172.18.0.3   │  │
│  │ resolv.conf:         │                          │ embedded DNS       │  │
│  │   nameserver         │                          │   127.0.0.11       │  │
│  │   127.0.0.11         │                          └────────────────────┘  │
│  └──────────────────────┘                                                  │
└────────────────────────────────────────────────────────────────────────────┘
```

#### (1) network namespace — the actual isolation

A kernel feature. Each namespace has its own **interfaces / routing table / iptables / `/etc/hosts` / `/etc/resolv.conf`**. One container = one namespace.

#### (2) veth pair — a virtual ethernet cable between namespaces

Two paired virtual NICs. Anything entering one end pops out the other. One end becomes the container's `eth0`; the other lives on the host as `vethXXXXXX`.

#### (3) bridge — an L2 software switch tying the host-side veth ends together

`docker-compose` creates `<project>_default` per project (shown as `br-xxxxx` on the host). A subnet (e.g. `172.18.0.0/16`) is assigned and each container gets an IP. **The bridge itself has an IP (`172.18.0.1`) — that's the containers' default gateway.**

```bash
docker network inspect <project>_default
```

#### (4) embedded DNS at `127.0.0.11`

Visible only inside each container's namespace. `/etc/resolv.conf` says `nameserver 127.0.0.11`. Resolving `postgres` → Docker answers `172.18.0.3`; unknown names are forwarded to the host DNS.

> [!warning] DNS works **only on user-defined networks**
> Two flavors of bridge:
> - **default bridge** (`docker0`): the fallback when `docker run` is called without `--network`. Containers can talk by IP, **but name resolution does not work**. This is why `--link` existed historically.
> - **user-defined network**: created by `docker network create` or implicitly by `docker-compose`. The embedded DNS maps name ↔ IP.
>
> `docker-compose` always creates user-defined networks, so DNS always works there. Plain `docker run` does not.

#### (5) Going to the internet = iptables MASQUERADE (SNAT)

A container's source IP is private (`172.18.0.2`); it cannot leave the host as-is. A `MASQUERADE` rule in the host's `nat POSTROUTING` chain rewrites the source IP to the host's. The reply is mapped back via [[conntrack]].

> [!example] Yes, conntrack stores a map
> A **hash table in kernel memory**.
> When the first packet goes out, an entry is created keyed by the 5-tuple (`proto, srcIP, srcPort, dstIP, dstPort`). If NAT is involved, both the **translation info** and the **expected reverse-direction tuple** of the reply are recorded in advance. When the reply arrives, that reverse tuple is looked up and the inverse translation is applied automatically.
>
> ```bash
> sudo conntrack -L            # or
> cat /proc/net/nf_conntrack
> ```

### 3. `extra_hosts: host.docker.internal:host-gateway` in detail

#### What this single line does

`extra_hosts` literally adds a line to the container's `/etc/hosts`:

```
host.docker.internal  <some_IP>
```

`host-gateway` is a magic keyword Docker replaces at runtime with "the host's IP as seen from the container" — usually the bridge gateway (`172.18.0.1`).

> [!tip] Why write it explicitly
> - **Docker Desktop (Mac/Windows)**: `host.docker.internal` is **provided out of the box**.
> - **Linux native Docker**: not provided by default. Adding `extra_hosts: ["host.docker.internal:host-gateway"]` makes the same compose file portable to Linux hosts.

#### macOS subtlety — host.docker.internal points to the **real Mac**, not the VM

> [!warning] Important correction
> Docker Desktop for Mac has two layers:
>   1. The **real macOS host** — where your `yarn dev` Node process binds to port `:3000`.
>   2. A **LinuxKit VM inside it** — where containers actually run; bridge/veth/iptables/embedded DNS all live in the VM.
>
> **`host.docker.internal` is wired by Docker Desktop to point at the real Mac, not the VM.** Its entire reason for existing is "let a container reach a service running on my Mac (e.g. an IDE dev server)".

#### How external traffic enters and gets routed

This repo's `nginx.conf` makes a deliberate choice — **every upstream is `host.docker.internal:PORT`**.

```nginx
upstream user-service     { server host.docker.internal:3000; }
upstream activity-service { server host.docker.internal:3001; }
upstream clutch-service   { server host.docker.internal:3002; }

server {
  listen 80;
  location /activity      { proxy_pass http://activity-service/; }
  location ~ ^/v1/fee     { proxy_pass http://clutch-service/;   }
  location /              { proxy_pass http://user-service/;     }
}
```

> [!note] `upstream user-service` is NOT a Docker DNS name
> The `user-service` token in `nginx.conf` is just an **nginx upstream block label** ��� unrelated to Docker's embedded DNS. `proxy_pass http://user-service/` expands to that block's content (`server host.docker.internal:3000`). nginx never uses container DNS at all in this repo.

#### Why "container" and "yarn dev" both work

> [!example] The fork happens at "who owns host:3000?", not inside nginx
>
> | Scenario | Who binds host(Mac):3000 |
> |---|---|
> | (1) user-service runs as a container | Docker Desktop, via `ports:3000:3000` — traffic is then hairpinned back into the VM container |
> | (2) `yarn dev` locally | The Mac's Node process binds directly |
>
> In both cases **arrival at Mac `:3000` is identical**, and nginx doesn't care which one it is. That's exactly why this repo did *not* use `server user-service:3000` (Docker DNS): so engineers can run a service locally via `yarn dev` and the nginx route still works.

### 4. External → nginx → service: the precise trace

> [!warning] "target is 3001" is ambiguous
> Two entry points:
> - **Case A — through nginx**: external hits `http://host/activity`, i.e. **port 80**. The `3001` is what nginx picks internally from the URL.
> - **Case B — straight to 3001**: if activity-service is also `ports:3001:3001`, the request bypasses nginx and goes directly to the container.
>
> If we're talking about nginx, **the entry port is 80**.

#### Case A — external client requests `/activity`

```
1. External client ─► macOS:80
                     │  Mac kernel: find the process bound to :80
                     │  → Docker Desktop (com.docker.backend)
                     ▼
2. Docker Desktop (gVisor/vpnkit) ─► forwards across the VM boundary into LinuxKit VM
                     │
                     ▼
3. VM iptables nat PREROUTING DNAT
   Rewrites destination from VM:80 → nginx container IP (e.g. 172.18.0.5):80
                     │  (L3/L4 routing in the kernel up to here)
                     ▼
4. VM kernel routing: that container IP lies across the bridge
   → bridge(br-xxxx) ─► veth pair ─► nginx eth0
                     │  (L2 transport)
                     ▼
5. nginx process receives on :80, parses HTTP
   location /activity matches → upstream activity-service
   = http://host.docker.internal:3001            ← L7 routing begins here
                     │
                     ▼
6. nginx opens a NEW connection out of the VM → Docker Desktop forwards to the Mac host
                     │
                     ▼
7. macOS :3001
   (a) if activity-service is a container: it is `ports:3001:3001` published,
       so Desktop hairpins back into the VM container
   (b) if it's `yarn dev`: the Mac's Node process receives it directly
                     │
                     ▼
8. Replies unwind via conntrack reverse-mappings recorded at each hop
```

> [!warning] Two common misconceptions, graded
> - "nginx looks at iptables and routes to 3001" → **❌**. iptables only carries the packet *to* nginx; once nginx has it, decisions are **L7 (HTTP location / proxy_pass)**.
> - "Since the activity container sits on the same bridge, nginx goes straight to it via veth/bridge" → **❌**. The upstream is `host.docker.internal:3001`, so the request **leaves the VM to the Mac and hairpins back** through the published port. Had it been `server activity-service:3001`, it would have gone bridge-direct.

#### Case B — external goes straight to `:3001`

```
External ─► macOS:3001 ─► Docker Desktop ─► VM iptables DNAT
                                       → activity container:3001
```

### 5. "But the Mac has no iptables — how does it know it's Docker?"

> [!example] It doesn't. **Docker has already grabbed the port**, so packets just land there
>
> When you publish `ports:80:80`, Docker Desktop's background process (`com.docker.backend`) **binds() to the Mac's port 80 like any normal server program** — no iptables involved.
>
> When an external packet arrives, the Mac kernel:
> 1. Looks at the destination port (`80`).
> 2. Looks up which process bound that port in its socket table → Docker Desktop.
> 3. Hands the packet to that process.
>
> ```bash
> sudo lsof -nP -iTCP:80 -sTCP:LISTEN
> # COMMAND            PID USER  ... NODE NAME
> # com.docker.backend ... ...   ... TCP  *:80 (LISTEN)
> ```
>
> Linux native = **kernel iptables DNAT**.
> Docker Desktop on macOS = **a userspace process bound to the port**.
>
> If a port isn't published, nothing binds it → `connection refused`.

### 6. veth/bridge vs iptables vs nginx — "routing" lives at two layers

> [!tip] "Routing" means different things at different layers
>
> | Layer | Done by | Decision input | Meaning |
> |---|---|---|---|
> | **L2 transport** | veth + bridge (kernel) | MAC, physical path | "the wire packets travel on" |
> | **L3/L4 routing** | iptables / routing table (kernel) | dst IP & port | "based on IP/port, send it where" |
> | **L7 routing** | nginx process | URL path, Host header, etc. | "based on HTTP content, send it where" |

So it's **not** "L7 does the routing." It's a **two-stage pipeline**: L3/L4 routing in the kernel delivers the packet to nginx, then L7 routing inside nginx decides the upstream.

VM entry sequence, restated:

1. Packet arrives at the VM (dst = VM interface:80).
2. **iptables nat PREROUTING DNAT** rewrites dst to nginx container IP:80 (L3/L4).
3. VM kernel routing: that container IP is across the bridge → **bridge → veth → nginx eth0** (L2 transport).
4. nginx receives on :80 → parses HTTP → matches `location /activity` → `proxy_pass` (**L7 begins**).

Your sentence — "once inside the VM, iptables forwards to nginx and L7 starts there" — is exactly this. The one nuance: "veth connected to a bridge enables networking" describes the **L2 transport path**, while **iptables / routing tables make the L3 decisions.** Keep transport (L2) and decision (L3) as separate ideas in your head and it stays clean.

### 7. One-picture summary

```
┌────────────────────────────────────────────────────────────────────┐
│                       External client                              │
└──────────────────────────────┬─────────────────────────────────────┘
                               │ TCP :80
                               ▼
       ┌─────────────── macOS host ──────────────────┐
       │  com.docker.backend bound to :80 receives    │   ← not iptables
       │  (visible via lsof)                          │
       └────────────────────┬─────────────────────────┘
                            │ gVisor/vpnkit forward
                            ▼
       ┌─────────── LinuxKit VM ─────────────────────┐
       │  iptables PREROUTING DNAT                   │   ← L3/L4
       │  VM:80 → nginx_ctn:80                       │
       │           │                                  │
       │           ▼ via bridge + veth                │   ← L2
       │     ┌─────────────────┐                      │
       │     │ nginx container │ ── HTTP parse        │
       │     │   location /act │ ── L7 routing  ──────┼─► host.docker.internal:3001
       │     └─────────────────┘                      │
       │           ▲                                  │
       │           │ replies: conntrack reverse-NAT   │
       └───────────┴──────────────────────────────────┘
                            ▲
                            │ hairpin ── Desktop forwards back into the VM
                            │            OR yarn dev on the Mac receives directly
                            ▼
                  activity-service container : 3001
                  or macOS Node process       : 3001
```

## Diagram

[[canvas/Docker_컨테이너와_네트워킹_완전_이해.canvas|Concept map]]
