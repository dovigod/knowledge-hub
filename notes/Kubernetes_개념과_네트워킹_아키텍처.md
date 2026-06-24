---
id: 019ec0c6-1c2b-7348-935a-deafa2cb6d8a
title: Kubernetes 개념과 네트워킹 아키텍처
topics:
  - kubernetes
  - networking
  - service-discovery
  - cni
  - kube-proxy
  - iptables
  - probe
tags:
  - kubernetes
  - k8s
  - docker
  - networking
  - service-discovery
  - cni
  - vxlan
  - bgp
  - kube-proxy
  - iptables
  - dnat
  - ebpf
  - cilium
  - liveness-probe
  - readiness-probe
  - infra
sources:
  - 019ec0c1-057c-7671-9fd3-5219d44dce6a
created_at: '2026-06-13T11:37:49.355Z'
updated_at: '2026-06-13T11:37:49.355Z'
---
> [!note] TL;DR
> [[Kubernetes]]는 "여러 머신 위에 흩어진 컨테이너 수십~수백 개를 선언적으로 자동 운영"하는 시스템이다. 핵심은 ① **Pod**(최소 배포 단위) ② **Service**(가상 IP로 흩어진 Pod에 접근) ③ **Deployment**(원하는 상태 유지) 세 가지. 네트워킹은 리눅스 원시 기능(`network namespace` + `veth` + `iptables`)을 그대로 물려받아, [[CNI]] 플러그인이 노드 간 Pod 통신을 책임지고 [[kube-proxy]]가 각 노드의 `iptables`/IPVS 체인(`KUBE-SERVICES → KUBE-SVC → KUBE-SEP`)에 동적으로 DNAT 규칙을 써넣어 무중단 로드밸런싱을 한다. [[Cilium]]/[[eBPF]]는 이 `iptables` 선형 탐색을 커널 내 O(1) 해시맵으로 갈아끼운 차세대 방식.

## 1. Docker만으로 부족한 순간 — 왜 [[Kubernetes]]인가

[[Docker]]는 "컨테이너 한 개를 만들고 실행"하는 도구다. 실제 서비스를 운영하면 곧 다음 상황에 부딪힌다.

- 트래픽이 몰리면 컨테이너를 10개로 늘려야 한다(한가하면 다시 줄여야 한다)
- 컨테이너가 죽으면 누군가 다시 살려야 한다(새벽 3시에도)
- 컨테이너 10개가 여러 서버에 흩어져 있으면 어느 서버에 띄울지 정해야 한다
- 새 버전 배포할 때 서비스 중단 없이 하나씩 교체해야 한다
- 컨테이너끼리 서로 찾아서 통신해야 한다(IP가 계속 바뀌는데)

이걸 사람이 손이나 셸 스크립트로 다 처리하기엔 양이 많고 실수가 잦다. 이 "여러 컨테이너를 자동으로 관리"하는 일을 **컨테이너 오케스트레이션**이라 부르고, [[Kubernetes]](줄여서 k8s)가 사실상의 표준이다.

> [!tip] 비유
> [[Docker]]는 "악기 한 대", [[Kubernetes]]는 "오케스트라 지휘자"다. 연주자(컨테이너)가 아무리 많아도 지휘자가 누가 언제 무엇을 연주할지 조율해 준다.

### Kubernetes가 해주는 핵심 일

| 기능 | 설명 |
|---|---|
| **자가 치유 (Self-healing)** | 컨테이너가 죽으면 자동으로 다시 띄움 |
| **오토스케일링** | 부하에 따라 컨테이너 개수를 자동으로 증감 |
| **스케줄링** | 여러 서버 중 자원이 남는 곳에 컨테이너를 배치 |
| **롤링 업데이트** | 무중단으로 새 버전 배포, 문제 시 자동 롤백 |
| **서비스 디스커버리 / 로드밸런싱** | 이름으로 찾고 트래픽을 고르게 분배 |
| **선언적 관리** | "컨테이너 3개 떠 있어야 함"이라고 선언하면 알아서 유지 |

마지막 "**선언적 관리(declarative)**"가 핵심 철학이다. Docker는 "이거 해, 저거 해"라고 명령(절차)하는 느낌이라면, k8s는 "최종 상태가 이래야 해"라고 [[YAML]]로 선언하면 현재 상태와 비교해 알아서 맞춰 준다.

### 핵심 객체 4가지

- **[[Pod]]**: k8s의 최소 배포 단위. 컨테이너 1개(또는 몇 개)를 묶은 것
- **[[Deployment]]**: "이 Pod를 몇 개 유지해 줘"를 선언(스케일링/롤링 업데이트 담당)
- **[[Service]]**: 흩어진 Pod들에 접근하는 고정 진입점(가상 IP + 로드밸런서)
- **[[Node]]**: Pod가 실제로 돌아가는 머신. 여러 Node를 묶은 게 [[Cluster]]

### 언제 필요한가?

- 서비스 1~2개, 트래픽 적음 → [[Docker Compose]]만으로 충분
- 마이크로서비스 15개 이상, 여러 서버, 무중단 배포, 트래픽 변동 큼 → [[Kubernetes]]가 빛을 발함

## 2. "nginx 띄우면 되는 거 아냐?" — 서비스 디스커버리의 진짜 문제

반은 맞다. [[nginx]]는 로드밸런싱(여러 백엔드에 트래픽 분배)은 잘 한다. **문제는 그 백엔드 목록을 누가 갱신하느냐**다.

```nginx
# 사람이 손으로 박아넣은 IP
upstream user_service {
  server 10.0.0.5:3000;
  server 10.0.0.6:3000;
  server 10.0.0.7:3000;
}
```

- 컨테이너 하나가 죽고 새로 뜨면 → IP가 바뀜 → nginx는 아직 죽은 IP로 트래픽을 보낸다
- 오토스케일링으로 3개 → 10개가 되면 → 새로 생긴 7개를 nginx가 모른다

nginx는 "분배"는 하지만 **"지금 살아있는 컨테이너가 누구인지"를 스스로 알아내지 못한다**. 누군가 설정 파일을 다시 쓰고 `nginx -s reload`를 해 줘야 하는데, 컨테이너가 수시로 죽고 살아나는 환경에서는 사람이 못 따라간다.

빠진 퍼즐 조각이 바로 **[[Service Discovery|서비스 디스커버리]]**: "이름표(예: `user-service`)를 주면, 지금 이 순간 살아있는 컨테이너들의 IP 목록을 자동으로 알려주는 시스템".

직접 만들려면 `nginx` + [[Consul]](또는 [[etcd]]) + `consul-template` 같은 조합이 필요하다. [[Kubernetes]]는 이걸 **내장으로** 준다. [[Service]] 객체를 만들면 `user-service`라는 고정 DNS 이름이 생기고, 뒤에 붙은 Pod들이 죽고 살아나도 k8s가 목록을 실시간으로 자동 갱신한다. 다른 컨테이너는 그냥 `http://user-service:3000`으로 부르면 알아서 살아있는 Pod로 연결된다. (외부 트래픽 받을 땐 k8s에서도 nginx를 [[Ingress]] 형태로 같이 쓴다.)

## 3. "여러 서버에 흩어진 컨테이너 — 어느 서버에 띄울지" — 스케줄링

이건 **[[Scheduling|스케줄링]]** 이야기다. 서버(머신)가 3대 있고 각각 CPU/RAM이 한정돼 있다. 컨테이너 10개(각각 자원 소비)를 어느 서버에 어떻게 나눠 배치할까? 한 서버에 몰면 터지고 나머지는 논다. 골고루 분산하려면 기준이 필요하다. 이게 "짐을 여러 트럭에 효율적으로 싣는 **bin-packing**" 문제와 같다.

```
[Node1: CPU 8, RAM 16GB]   [Node2: CPU 8, RAM 16GB]   [Node3: CPU 8, RAM 16GB]
       ↑                            ↑                             ↑
       └──── Pod(2,4) ────┐         │                             │
                          ├──── Pod(4,8) ────┐                    │
                          │                  ├──── Pod(1,2) ──────┤
                          │                  │                    │
            (스케줄러가 자원 여유 보고 자동 배치)
```

[[Docker]]만 쓰면 네가 직접 각 서버에 SSH로 접속해서 `docker run`을 손으로 결정/실행해야 한다. [[Kubernetes]]의 [[kube-scheduler|스케줄러]]가 이걸 자동으로 한다.

```yaml
# Pod의 자원 요구만 선언하면
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
# 스케줄러가 자원 남는 노드를 찾아 배치
```

`affinity`/`anti-affinity` 같은 규칙도 줄 수 있다("이 두 Pod는 같은 노드에 두지 마").

## 4. [[Docker Compose]] vs [[Kubernetes]]

| 항목 | Docker Compose | Kubernetes |
|---|---|---|
| **대상 범위** | 한 대 머신 | 여러 머신(클러스터) |
| **용도** | 로컬 개발 | 운영 대규모 |
| **컨테이너 죽으면** | `restart:always` 단순 재시작 | 자가치유 + 다른 서버 재배치 |
| **스케일링** | 수동(한 머신 안) | 자동(여러 서버 분산) |
| **로드밸런싱** | 거의 없음 | Service 내장 |
| **무중단 배포** | 직접 스크립트 | 롤링 업데이트 + 자동 롤백 내장 |
| **학습 난이도** | 쉬움 | 어려움 |

한 줄로: **[[Docker Compose]]=한 컴퓨터에서 컨테이너 여러 개 띄우는 도구 / [[Kubernetes]]=여러 컴퓨터를 묶어 컨테이너 수백 개를 자동 운영하는 시스템**. Compose에도 같은 네트워크 안 이름 기반 디스커버리는 있지만 **한 머신 안에서만** 작동한다.

> [!example] "서버 = 컴퓨터인가? 로컬엔 대응 없겠군"
> 맞다. 서버=컴퓨터(물리 서버든 클라우드 가상머신이든). k8s 용어로 **[[Node]]**. 여러 Node를 묶은 게 **[[Cluster]]**. 다만 "로컬엔 대응 없다"는 반만 맞다 — [[minikube]], [[kind]], [[Docker Desktop]](Enable Kubernetes)으로 **노트북 1대를 1-Node 클러스터**처럼 만들 수 있다. Node가 1개뿐이라 분산 스케줄링의 묘미는 못 보지만, Pod/Service/Deployment/자가치유/롤링 ��데이트 개념은 똑같이 실습 가능.

## 5. "Pod 늘어나면 라우팅 재시작 필요?" — 절대 아니다

`nginx` 수동 방식과 [[Kubernetes]]의 결정적 차이가 여기다.

- `nginx`: 설정 파일 수정 → `nginx -s reload`(설정 다시 읽는 행위가 명시적으로 일어남, 무중단이긴 함)
- `k8s`: **재시작 없음.** 커널의 라우팅 테이블 데이터만 한 줄 고친다.

### 전체 그림: 컨트롤 플레인 + 워커 노드

```
┌────────────────── Control Plane (두뇌) ──────────────────┐
│                                                          │
│  ┌────────────┐  ┌──────┐  ┌─────────────┐  ┌─────────┐ │
│  │ API Server │←→│ etcd │  │ Controllers │  │Scheduler│ │
│  │   (정문)    │  │ (DB) │  │ (감시/조정)  │  │ (배치)  │ │
│  └─────┬──────┘  └──────┘  └─────────────┘  └─────────┘ │
└────────┼─────────────────────────────────────────────────┘
         │ watch (gRPC stream)
   ┌─────┴─────────────────────────────┐
   ↓                                   ↓
┌──────────── Worker Node 1 ─────┐  ┌──────────── Worker Node 2 ─────┐
│ kubelet  kube-proxy   Pods...  │  │ kubelet  kube-proxy   Pods...  │
│  (Pod   (iptables/                │  │                              │
│   생사)  IPVS 갱신)                │  │                              │
└────────────────────────────────┘  └────────────────────────────────┘
```

### 단계별 흐름

**1단계: 새 Pod 뜨면 → [[Endpoints|EndpointSlice]]에 자동 추가**

선언적 상태 관리의 핵심은 **[[Reconciliation Loop|reconciliation loop]]**: 컨트롤러는 "원하는 상태 vs 현재 상태"를 비교해 다르면 조정한다. `replicas: 3` 같은 원하는 상태가 [[etcd]]에 저장되고, 새 Pod가 IP를 받으면 [[kubelet]]이 `Ready`를 API Server에 보고한다. `EndpointSlice` 컨트롤러가 이걸 감지해 엔드포인트 목록에 자동 추가한다.

> [!note] watch 메커니즘
> 폴링(polling)이 아니라 **"바뀌면 알려줘" 구독(watch)**이라 밀리초 단위 통보다. API Server가 길게 열린 gRPC 스트림으로 변경 이벤트를 푸시한다.

**2단계: kube-proxy가 변화 감지 → 라우팅 규칙 갱신**

각 노드의 [[kube-proxy]]도 API Server에 watch를 걸어 둔다. `EndpointSlice`가 바뀌면 즉시 통보받아 자기 노드의 라우팅 규칙을 다시 쓴다.

**3단계: 커널 레벨 규칙 테이블 수정 — iptables의 정체**

Service IP는 **어디에도 실제로 존재하지 않는 가상 IP(VIP)**다. 컨테이너가 VIP로 요청하면 리눅스 커널의 `netfilter`/`iptables` 규칙이 진짜 Pod IP로 바꿔치기한다 = **[[DNAT]](목적지 NAT)**.

kube-proxy가 만든 규칙은 대략:
> "목적지가 VIP면 1/3 확률로 Pod1, 1/3로 Pod2, 1/3로 Pod3로 DNAT"

**로드밸런싱이 애플리케이션 레벨이 아니라 커널 패킷 처리 단계에서 일어난다.**

> [!tip] 왜 재시작 불필요?
> `nginx`는 user space 프로세스라 설정을 reload해야 하지만, `iptables` 규칙은 커널이 갖고 있는 테이블 데이터일 뿐이라 줄 하나 추가/삭제하면 **다음 패킷부터** 새 규칙대로 처리된다. "다시 읽는 행위" 자체가 없다.
>
> 비유: `nginx reload`=메뉴판 새로 인쇄해 배포 / `iptables` 갱신=화이트보드 한 줄 고치기.

참고: 규칙이 수천 개로 늘면 [[iptables]]는 선형 탐색이라 느려져 대규모는 [[IPVS]](해시 테이블 기반)를 쓴다. 원리는 같고 성능만 좋아진 버전.

> [!example] "Docker에선 노드가 물리 컴퓨터 한 대?"
> **노드(Node)는 Docker 단독 개념이 아니라 오케스트레이터 개념**이다. 그냥 `docker run`은 "노드"라는 말을 거의 안 쓴다(머신 1대). [[Docker Swarm]]/Kubernetes에서 머신 1대=1 노드. 노드=컨테이너가 도는 머신 한 대가 맞지만 **물리 컴퓨터로 한정되지 않는다**(클라우드 가상머신일 수도, 실무에선 가상머신이 더 흔하다).

## 6. Docker 컨테이너와의 구조적 관계 — 같은 리눅스 뿌리

직감대로 [[Kubernetes]] 네트워킹은 [[Docker]]가 쓰는 리눅스 기능들을 그대로 물려받아 확장한 거다.

### 공통 뿌리: [[network namespace]] + `veth` + `iptables`

이 셋은 리눅스 커널 기능이지 Docker/k8s 발명품이 아니다.

- **network namespace(`netns`)**: 네트워크 스택(인터페이스, 라우팅 테이블, `iptables` 규칙)을 통째로 격리한 독립 공간
- **`veth` pair**: 두 namespace를 잇는 "가상 랜선"
- **`iptables`**: 각 namespace마다 **독립적으로** 존재하는 규칙 테이블

> [!warning] 핵심
> `iptables`는 `netns`에 종속된다. **`netns`가 다르면 `iptables`도 별개**.

### Docker의 네트워크 구조

```
┌──────────────── Host (root netns) ────────────────┐
│                                                   │
│           ┌──── docker0 (bridge) ────┐            │
│           │                          │            │
│      veth0│                          │veth1       │
│        ──┴── ╳ ─────────────── ╳ ──┴──            │
│         │                            │            │
│  ┌──────┴──────┐              ┌──────┴──────┐    │
│  │ Container A │              │ Container B │    │
│  │ (netns A)   │              │ (netns B)   │    │
│  │  eth0 IP    │              │  eth0 IP    │    │
│  └─────────────┘              └─────────────┘    │
│                                                   │
│   외부로 나갈 때: iptables MASQUERADE (SNAT)       │
│   포트 매핑 -p 8080:80: iptables DNAT             │
└───────────────────────────────────────────────────┘
```

→ Docker도 이미 `DNAT`/`SNAT`을 `iptables`로 쓰고 있던 거다.

### k8s가 올린 세 가지 변화

| 변화 | Docker | Kubernetes |
|---|---|---|
| **netns 단위** | 컨테이너마다 1개 | **Pod마다 1개** (Pod 안 컨테이너들은 netns 공유, `pause` 컨테이너가 들고 있음 → Pod 내 컨테이너는 `localhost`로 통신) |
| **노드 간 연결** | `docker0` 단일 브리지(한 머신) | **CNI 플러그인**([[Calico]]/[[Flannel]] 등) — 다른 노드의 Pod IP끼리 직접 통신 |
| **Service VIP의 DNAT** | 없음 | **호스트 root netns의 iptables**에서 일어남(`kube-proxy`가 여기 써넣음) |

> [!note] 패킷 여정
> Pod A netns에서 VIP로 출발 → `veth` 타고 호스트 root netns 진입 → 여기서 `iptables` DNAT으로 실제 Pod IP로 변경 → CNI 타고 목적지 Pod 도착(다른 노드여도 OK).

Pod의 netns 안 `iptables`는 보통 거의 비어있고, **호스트 netns의 `iptables`에 `KUBE-SERVICES`/`KUBE-SVC`/`KUBE-SEP` 체인이 들어간다**.

한 문장: **둘 다 같은 리눅스 원시 기능(`netns`+`veth`+`iptables` DNAT)을 쓰는데, Docker는 한 머신에서 컨테이너 격리+포트 매핑까지만, k8s는 그 위에 (1) Pod 단위 netns (2) 여러 노드를 잇는 CNI (3) `kube-proxy`가 호스트 `iptables`에 써넣는 동적 Service 로드밸런싱을 얹은 것.**

## 7. 봉우리 1 — [[CNI]]가 노드 간 Pod 통신을 실제로 어떻게?

### 왜 문제인가

Pod IP(`10.244.x.x` 같은 대역)는 k8s 내부 가상 대역이다. 노드를 잇는 **물리 네트워크(스위치/라우터)는 이 대역을 모른다**(노드의 실제 IP `192.168.1.x`만 안다). 이 간극을 메우는 게 CNI의 임무. 두 갈래가 있다.

### 방식 A: 오버레이(VXLAN) — "봉투에 넣어 부치기"

대표: [[Flannel]](기본), [[Calico]] VXLAN 모드.

물리망이 모르는 Pod 패킷을 **물리망이 아는 노드 패킷으로 한 번 감싸서**(encapsulation) 보낸다.

```
원본 패킷: [src=10.244.1.5  dst=10.244.2.7  payload]
                          ↓ encapsulate
바깥 봉투: [src=192.168.1.10 dst=192.168.1.20  UDP:8472  VXLAN-header  | 원본 ]
                          ↓ 물리망은 이걸 보고 라우팅
                  목적지 노드 도착 → 봉투 까서 원본만 꺼냄 → Pod B 전달
```

흐름:
1. Node1의 `flannel.1`(VXLAN 장치)이 가로채 "10.244.2.7은 Node2" 조회(FDB)
2. UDP 봉투에 넣어 발송
3. Node2 `flannel.1`이 개봉 → Pod B에 전달

| 장점 | 단점 |
|---|---|
| 물리망이 멍청해도 무조건 작동 | 캡슐화 CPU 오버헤드 |
| 클라우드 [[VPC]] 넘나듦 | MTU 감소(~50바이트) |
| | 디버깅 어려움(`tcpdump`로 보면 봉투에 가려짐) |

### 방식 B: 라우팅(L3/BGP) — "길 안내판 세우기"

대표: [[Calico]] BGP, [[Cilium]], [[Flannel]] `host-gw`.

봉투 없이, **각 노드가 자기 Pod 대역을 광고**해 라우팅 테이블로 직접 보낸다.

```
Node1 라우팅 테이블:
  10.244.2.0/24  via  192.168.1.20  (Node2)
  10.244.3.0/24  via  192.168.1.30  (Node3)

→ Pod 패킷 그대로 [src=10.244.1.5 dst=10.244.2.7] 발송
→ 물리망은 라우팅 테이블 보고 Node2로 넘김
```

[[BGP]](Calico는 [[BIRD]] 데몬)로 노드끼리 경로를 자동 동기화한다. **패킷은 봉투 없이 원본 그대로**.

| 장점 | 단점 |
|---|---|
| 거의 네이티브 속도 | 물리 네트워크가 똑똑해야 함(L3 도달성/BGP 지원) |
| MTU 문제 없음 | 다른 서브넷에 흩어지면 설정 까다로움 |
| 디버깅 쉬움 | |

한 줄: **둘 다 "물리망이 모르는 Pod IP를 노드 간 전달" 문제를 푸는데, 오버레이는 포장해서 우회, 라우팅은 길 알려줘 직행.**

## 8. 봉우리 2 — [[kube-proxy]]의 `iptables` 체인 해부

### 전체 계층

```
패킷 진입(PREROUTING/OUTPUT)
        ↓
┌───────────────────┐
│  KUBE-SERVICES    │  ← 어느 Service인가 분류 (VIP로 매칭)
└────────┬──────────┘
         ↓
┌───────────────────┐
│  KUBE-SVC-XXX     │  ← 로드밸런싱: 확률 분배
└────────┬──────────┘
         ↓
┌───────────────────┐
│  KUBE-SEP-YYY     │  ← 실제 DNAT (VIP → 진짜 Pod IP)
└───────────────────┘
```

`SVC`=Service, `SEP`=Service EndPoint(Pod 하나).

### ① `KUBE-SERVICES` — 교통정리

모든 Service VIP가 한 줄씩 들어 있다.

```
-d 10.96.0.10/32 -p tcp --dport 3000 -j KUBE-SVC-ABC
-d 10.96.0.11/32 -p tcp --dport 80   -j KUBE-SVC-DEF
...
```

어느 Service 행이냐만 판별해 분기한다.

### ② `KUBE-SVC-XXX` — 로드밸런서(확률 분배)

`statistic` 모듈로 확률 기반 분배.

```
-m statistic --mode random --probability 0.33333  -j KUBE-SEP-001
-m statistic --mode random --probability 0.50000  -j KUBE-SEP-002
                                                  -j KUBE-SEP-003   (마지막은 무조건)
```

> [!tip] 확률이 0.33 → 0.5 → 1.0로 변하는 이유
> 순차적으로 걸러지기 때문이다. 첫 줄에서 1/3 통과, 남은 2/3 중 1/2가 두 번째 = 전체의 1/3, 나머지 1/3은 세 번째. **정확히 3등분**.

### ③ `KUBE-SEP-YYY` — 실제 DNAT

SEP 하나 = Pod 하나.

```
(a) -s 10.244.1.5/32 -j KUBE-MARK-MASQ            ← 헤어핀 대비 SNAT 표시
(b) -p tcp -j DNAT --to-destination 10.244.1.5:3000   ← 가상 IP → 진짜 Pod IP
```

(b)가 **DNAT 실물**. 가상 VIP를 진짜 Pod IP로 바꿔치기하는 줄이다.

### 빠진 조각 2개

**(1) `KUBE-MARK-MASQ` / `KUBE-POSTROUTING` (SNAT)**

DNAT으로 목적지만 바꾸면 응답이 엉뚱한 데로 갈 수 있다(헤어핀, 클러스터 밖 출발). 그래서 마킹한 패킷은 `KUBE-POSTROUTING`에서 `MASQUERADE`로 출발지도 바꾼다.

**(2) [[conntrack]] (연결 추적)**

갈 때 DNAT한 걸 커널이 기억해, **응답 패킷이 오면 자동으로 역변환**(Pod IP → VIP 출발지). DNAT은 첫 패킷에만 적용되고 나머지는 conntrack이 처리 → 빠르고 세션 일관성 유지.

> [!note] 두 봉우리가 만나는 지점
> `KUBE-SEP`에서 DNAT으로 "목적지=다른 노드 Pod IP"가 정해지면, 실제 운반은 봉우리 1의 [[CNI]](오버레이/라우팅)가 한다. **iptables=어느 Pod로 보낼지 정하고 주소 변환, CNI=정해진 Pod까지 물리적 배달**.

## 9. Probe — Pod의 생사·준비 판정

### 세 가지 다른 질문

| Probe | 질문 | 실패 시 동작 |
|---|---|---|
| **liveness** | "살아있어?" | 컨테이너를 죽이고 재시작 |
| **readiness** | "트래픽 받을 준비 됐어?" | Service 엔드포인트에서 **제외**(재시작은 X) |
| **startup** | "다 켜졌어?" | 켜질 때까지 위 둘 보류 |

**liveness ≠ readiness** 구분이 핵심이다.

### readiness probe — 미뤄둔 그것

앞서 "Pod가 떠도 진짜 준비됐는지 확인하고 엔드포인트 추가"라 했던 그 확인이 readiness다. 컨테이너가 떠도 DB 풀, 캐시 워밍, 설정 로딩에 시간이 걸려 즉시 트래픽 처리가 불가능하다.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  periodSeconds: 5
  failureThreshold: 3
```

연결 흐름:
```
readiness 통과
   → kubelet이 API Server에 Ready 보고
   → EndpointSlice 컨트롤러가 Pod IP를 엔드포인트에 추가
   → kube-proxy가 iptables KUBE-SVC에 줄 추가
   → 트래픽 흐름 시작

readiness 실패
   → 엔드포인트에서 제거
   → iptables 줄 삭제
   → 트래픽 안 감 (컨테이너는 안 죽임)
```

즉 **readiness=엔드포인트 명단의 입출입 스위치**. 롤링 업데이트 무중단의 이유도 이것이다(새 Pod readiness 통과 후에야 트래픽 받음).

### liveness probe

"좀비 됐냐"(데드락 등)를 검사한다. 실패 시 [[kubelet]]이 컨테이너를 죽이고 재시작한다(반복되면 `CrashLoopBackOff` 상태).

### startup probe

부팅에 2분 걸리는 무거운 앱에 liveness만 걸면 부팅 중에 영원히 재시작 루프에 빠진다. startup probe는 **다 켜질 때까지 liveness/readiness를 비활성화**하는 보호막이다(`failureThreshold × periodSeconds`로 최대 대기 시간 산정). 성공하면 그때부터 나머지 probe 가동.

### probe 방식 4가지

- `httpGet`: HTTP 200~399면 성공
- `tcpSocket`: 포트가 열리면 성공
- `exec`: 명령 exit code 0이면 성공
- `grpc`: gRPC health check

> [!warning] 실무 흔한 사고
> 1. **liveness에 의존성 체크를 넣음** → DB가 잠깐 느려질 때 멀쩡한 컨테이너들이 줄줄이 재시작되는 *cascading failure*. liveness는 "나 자신이 좀비냐"만, **의존성은 readiness로**.
> 2. **probe 없이 배포** → 준비 안 된 Pod로 트래픽 가서 에러.
> 3. **initialDelay/threshold가 너무 빡빡** → 정상인데 오판해서 재시작 루프.

## 10. [[eBPF]] 기반 [[Cilium]] — `iptables`를 아예 안 쓰는 차세대 방식

### 왜 iptables를 버리나

1. **선형 탐색 O(n)**: `KUBE-SERVICES`를 위에서부터 하나씩 비교. Service 수천 개면 규칙 수만~수십만 줄
2. **갱신 비용**: Pod 하나 바뀌면 체인 재작성이 무거움
3. **user/kernel 경계, conntrack 오버헤드**

작은 클러스터는 OK지만 수천 서비스 규모에서 병목이 된다.

### [[eBPF]]란

"커널 안에서 안전하게 돌릴 수 있는 작은 프로그램" — **"커널을 위한 JavaScript"**. 재컴파일/재부팅 없이 커널 hook에 검증된 프로그램을 꽂는다.

```
┌──────────────── Linux Kernel ────────────────┐
│                                              │
│   [verifier] ← eBPF 코드 검증(무한루프/메모리 안전) │
│        ↓                                     │
│   [hook 지점] XDP, tc, socket, kprobe...     │
│        ↓                                     │
│   [eBPF program] 실행 (커널 안, 매우 빠름)      │
│        ↕                                     │
│   [eBPF map] 해시 테이블 ← "VIP→Pod" 저장      │
└──────────────────────────────────────────────┘
```

특성:
- 커널 안 실행(빠름)
- verifier가 무한루프/메모리 안전을 강제 검사(안전)
- eBPF map(해시 테이블)에 "VIP→Pod" 저장해 **O(1) 조회**
- hook 지점(XDP/tc/socket)에 꽂아 패킷 경로 어디서든 가로채기 가능

### [[Cilium]]이 하는 일

`kube-proxy`를 통째로 대체한다.

| 기존 [[iptables]] | [[Cilium]]/[[eBPF]] |
|---|---|
| O(n) 선형 탐색 | O(1) 해시 조회 |
| 확률 분배 → DNAT | map에서 즉시 목적지 선택 |
| 규칙 수만큼 느려짐 | 규모 무관 일정 |
| 체인 재작성 | map 한 줄만 갱신 |

특히 **소켓 레벨 로드밸런싱(socket LB)**: 내부(east-west) 통신은 `connect()` 시스템 콜 단계에서 목적지를 바로 Pod IP로 꽂아 **패킷마다 DNAT을 할 필요가 없다**. conntrack 부담도 감소, 거의 오버헤드 0.

### 장점 정리

- **O(1) 해시맵** → 규모 무관 일정
- **map 항목 1줄 수정**으로 즉시 갱신
- **소켓 레벨 LB**(연결당 1회) → 패킷마다 DNAT 불필요
- **Identity(라벨) 기반 보안**: Pod IP가 바뀌어도 정책 유지, L7까지 적용
- **[[Hubble]] 관측성**: eBPF 기반 트래픽 가시화

단점: **비교적 최신 커널 필요**.

> [!note] 봉우리 1과의 관계
> [[Cilium]]은 [[CNI]] 역할 + `kube-proxy` 역할 + [[NetworkPolicy]]를 **전부 eBPF로 한 번에 처리하는 올인원**.

한 줄: **iptables는 규칙을 선형으로 훑는 옛 방식이라 대규모에서 느려진다. eBPF는 커널 안에서 해시맵으로 O(1) 처리하고 소켓 레벨에서 LB하며, Cilium이 이걸로 kube-proxy + CNI + 보안을 한 번에 대체한다.**

## 다이어그램

[[canvas/Kubernetes_개념과_네트워킹_아키텍처.canvas|개념도]]

---

## English

> [!note] TL;DR
> [[Kubernetes]] is the system for "running tens to hundreds of containers spread across many machines, declaratively and automatically." Three core objects: **Pod** (unit of deployment), **Service** (virtual IP fronting a set of Pods), **Deployment** (maintains desired state). Networking inherits Linux primitives — `network namespace` + `veth` + `iptables` — with [[CNI]] plugins handling cross-node Pod traffic and [[kube-proxy]] writing dynamic `iptables`/IPVS chains (`KUBE-SERVICES → KUBE-SVC → KUBE-SEP`) on each node for zero-downtime load balancing. [[Cilium]]/[[eBPF]] is the next-gen replacement that swaps that linear-scan `iptables` for an in-kernel O(1) hash map.

### 1. Where Docker alone falls short — why [[Kubernetes]]

[[Docker]] is the tool for "building and running one container." In production you soon hit:

- Traffic spikes → grow from 1 container to 10 (and shrink back when quiet)
- A container dies → someone must restart it (even at 3am)
- 10 containers across multiple servers → who decides where each lands?
- New version → must roll out without downtime
- Containers must find each other to talk — but IPs keep changing

Doing this by hand or with shell scripts doesn't scale. This "automatic management of many containers" is **container orchestration**, and [[Kubernetes]] (k8s) is the de-facto standard.

> [!tip] Analogy
> [[Docker]] is "one instrument," [[Kubernetes]] is the "orchestra conductor." No matter how many performers (containers), the conductor coordinates who plays what when.

#### What Kubernetes gives you

| Feature | Description |
|---|---|
| **Self-healing** | Dead containers auto-restart |
| **Autoscaling** | Container count auto-grows/shrinks with load |
| **Scheduling** | Picks the server with free resources |
| **Rolling updates** | Zero-downtime deploys, auto-rollback on failure |
| **Service discovery / load balancing** | Find peers by name, distribute traffic |
| **Declarative management** | Declare "3 must be running" → it maintains that state |

The last one — **declarative management** — is the core philosophy. Docker feels imperative ("do this, do that"); k8s says "the final state should look like X" in [[YAML]] and the system reconciles current → desired.

#### Four core objects

- **[[Pod]]**: smallest deployable unit (one or a few containers)
- **[[Deployment]]**: "keep this many replicas of this Pod" (scaling + rolling updates)
- **[[Service]]**: stable entry point fronting scattered Pods (virtual IP + load balancer)
- **[[Node]]**: a machine where Pods actually run; many Nodes = a [[Cluster]]

#### When you need it

- 1–2 services, low traffic → [[Docker Compose]] is enough
- 15+ microservices, multiple servers, zero-downtime deploys, spiky traffic → [[Kubernetes]] earns its keep

### 2. "Can't we just run nginx?" — the real service-discovery problem

Half right. [[nginx]] does load balancing fine. **The real question is: who keeps the backend list fresh?**

```nginx
# Hand-typed IPs
upstream user_service {
  server 10.0.0.5:3000;
  server 10.0.0.6:3000;
  server 10.0.0.7:3000;
}
```

- A container dies and respawns → new IP → nginx still sends traffic to a dead one
- Autoscaling 3 → 10 → nginx never learns about the 7 new pods

nginx distributes but **cannot discover** who's alive right now. Someone has to rewrite config and `nginx -s reload`, which a human can't keep up with when containers churn constantly.

The missing piece is **[[Service Discovery]]**: "given a name (e.g. `user-service`), tell me the live IPs right now."

Building it yourself = `nginx` + [[Consul]] (or [[etcd]]) + `consul-template`. [[Kubernetes]] gives you this **built in**. Create a [[Service]] and you get `user-service` as a stable DNS name; as Pods come and go, k8s auto-updates the list in real time. Other containers just call `http://user-service:3000`. (External traffic still typically uses nginx — as [[Ingress]] — in front of it.)

### 3. "Which server to put it on?" — scheduling

This is **[[Scheduling]]**. With 3 servers each capped on CPU/RAM, where do you place 10 containers? Piling them on one fries that node and idles the rest. You need a placement strategy — classic **bin-packing**.

```
[Node1: CPU 8, RAM 16GB]   [Node2: CPU 8, RAM 16GB]   [Node3: CPU 8, RAM 16GB]
       ↑                            ↑                             ↑
       └──── Pod(2,4) ────┐         │                             │
                          ├──── Pod(4,8) ────┐                    │
                          │                  ├──── Pod(1,2) ──────┤
                          │                  │                    │
                  (scheduler watches free capacity & places)
```

With plain [[Docker]] you SSH into each server and decide manually. The [[kube-scheduler]] does this automatically:

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
# → scheduler finds a node with capacity and places it
```

You can layer in `affinity` / `anti-affinity` ("never co-locate these two Pods").

### 4. [[Docker Compose]] vs [[Kubernetes]]

| | Docker Compose | Kubernetes |
|---|---|---|
| **Scope** | One machine | Many machines (cluster) |
| **Use case** | Local dev | Production at scale |
| **Container dies** | `restart:always` | Self-heal + reschedule to another node |
| **Scaling** | Manual, single host | Auto, distributed |
| **Load balancing** | Basically none | Service built in |
| **Zero-downtime deploy** | Custom scripts | Built-in rolling update + auto-rollback |
| **Learning curve** | Easy | Hard |

One-liner: **Compose = "run several containers on one machine" / Kubernetes = "run hundreds of containers across many machines, automatically."** Compose has name-based discovery too, but **only within one host**.

> [!example] "Server = computer? Then nothing local maps to it?"
> Right — server = computer (physical or VM). k8s calls it a **[[Node]]**; many Nodes = a **[[Cluster]]**. Only half right that nothing local maps: [[minikube]], [[kind]], or [[Docker Desktop]]'s built-in Kubernetes turn your laptop into a single-Node cluster. You miss the distributed-scheduling part but Pod/Service/Deployment/self-healing/rolling-update concepts all work identically.

### 5. "Does adding Pods require restarting the router?" — No

This is the decisive break from manual `nginx`.

- `nginx`: edit config → `nginx -s reload` (explicit reload step; graceful but it does happen)
- `k8s`: **no restart, ever.** A single line in the kernel routing table changes.

#### The big picture: control plane + worker nodes

```
┌─────────────── Control Plane (brain) ────────────────┐
│                                                      │
│  ┌────────────┐  ┌──────┐  ┌─────────────┐  ┌───────┐│
│  │ API Server │←→│ etcd │  │ Controllers │  │Sched. ││
│  │  (gateway) │  │ (DB) │  │ (reconcile) │  │       ││
│  └─────┬──────┘  └──────┘  └─────────────┘  └───────┘│
└────────┼─────────────────────────────────────────────┘
         │ watch (gRPC stream)
   ┌─────┴─────────────────────────────┐
   ↓                                   ↓
┌──────────── Worker Node 1 ─────┐  ┌──────────── Worker Node 2 ─────┐
│ kubelet  kube-proxy   Pods...  │  │ kubelet  kube-proxy   Pods...  │
│  (Pod    (iptables/                │  │                              │
│   lifecycle)  IPVS)                │  │                              │
└────────────────────────────────┘  └────────────────────────────────┘
```

#### Step-by-step flow

**Step 1: new Pod → controller adds it to [[Endpoints|EndpointSlice]]**

The heart of declarative state is the **[[Reconciliation Loop]]**: controllers compare *desired vs current* and act when they differ. Desired state (e.g. `replicas: 3`) lives in [[etcd]]. When a new Pod gets an IP, [[kubelet]] reports `Ready` to the API Server, and the `EndpointSlice` controller appends that Pod's IP to the endpoint list.

> [!note] watch mechanism
> Not polling — a long-lived gRPC "tell me when something changes" subscription. Updates arrive in milliseconds.

**Step 2: kube-proxy sees the change → rewrites routing rules**

Each node's [[kube-proxy]] keeps its own watch on the API Server. The moment `EndpointSlice` changes, kube-proxy updates that node's routing tables.

**Step 3: kernel-level rule update — what `iptables` actually is**

The Service IP is **a virtual IP that doesn't exist on any interface**. When a container sends to the VIP, Linux's `netfilter`/`iptables` rewrites the destination to a real Pod IP — **[[DNAT]] (destination NAT)**.

kube-proxy's rules read roughly:
> "If destination = VIP, with 1/3 probability go to Pod1, 1/3 to Pod2, 1/3 to Pod3, with destination rewritten."

**Load balancing happens at the kernel packet level, not the app level.**

> [!tip] Why no restart needed
> `nginx` is a user-space process that has to re-read config. `iptables` rules are kernel data tables — change a row and the **very next packet** uses the new rule. There is no "re-read" step.
>
> Analogy: `nginx reload` = reprint and redistribute the menu / `iptables` update = erase one line on the whiteboard and write a new one.

Side note: with thousands of rules, `iptables`'s linear matching gets slow, so large clusters use [[IPVS]] (hash-table based) — same idea, faster lookup.

> [!example] "In Docker, is a node just one physical computer?"
> **"Node" is an orchestrator concept, not a plain-Docker concept.** Vanilla `docker run` doesn't really use the word (it's just "the host"). In [[Docker Swarm]] or Kubernetes, one machine = one Node. A Node *is* the machine running containers, but **not necessarily physical** — a cloud VM is by far the more common case.

### 6. Relation to Docker containers — same Linux roots

Right intuition: k8s networking inherits the same Linux primitives Docker uses, then extends them.

#### Shared substrate: [[network namespace]] + `veth` + `iptables`

These are Linux kernel features, not Docker/k8s inventions.

- **network namespace (`netns`)** — isolated network stack (interfaces, routing table, `iptables` rules)
- **`veth` pair** — a "virtual cable" linking two namespaces
- **`iptables`** — rule table that lives **inside each namespace independently**

> [!warning] Key
> `iptables` is scoped to `netns`. **Different `netns` → different `iptables`**.

#### Docker's network shape

```
┌──────────────── Host (root netns) ────────────────┐
│                                                   │
│           ┌──── docker0 (bridge) ────┐            │
│           │                          │            │
│      veth0│                          │veth1       │
│        ──┴── ╳ ─────────────── ╳ ──┴──            │
│         │                            │            │
│  ┌──────┴──────┐              ┌──────┴──────┐    │
│  │ Container A │              │ Container B │    │
│  │ (netns A)   │              │ (netns B)   │    │
│  │  eth0 IP    │              │  eth0 IP    │    │
│  └─────────────┘              └─────────────┘    │
│                                                   │
│   Egress to internet: iptables MASQUERADE (SNAT)  │
│   Port mapping -p 8080:80: iptables DNAT          │
└───────────────────────────────────────────────────┘
```

→ Docker was already doing `DNAT`/`SNAT` via `iptables`.

#### What k8s adds

| | Docker | Kubernetes |
|---|---|---|
| **netns granularity** | per container | **per Pod** (containers inside a Pod share a netns — the `pause` container owns it, so they talk over `localhost`) |
| **Cross-node link** | a single `docker0` bridge (one host) | **CNI plugin** ([[Calico]]/[[Flannel]]/…) — Pods on different nodes reach each other by Pod IP |
| **Service VIP DNAT** | n/a | Happens in the **host root netns iptables** (where `kube-proxy` writes rules) |

> [!note] Packet journey
> Pod A's netns sends to a VIP → via `veth` into the host root netns → host `iptables` DNATs to a real Pod IP → CNI delivers to the destination Pod (same node or another).

Inside a Pod's netns the `iptables` table is mostly empty; the **host's `iptables` carries the `KUBE-SERVICES` / `KUBE-SVC` / `KUBE-SEP` chains.**

One sentence: **Both use the same Linux primitives (`netns` + `veth` + `iptables` DNAT). Docker stops at "isolate containers on one host + map ports." k8s layers on (1) per-Pod netns, (2) CNI that spans many nodes, and (3) `kube-proxy` writing dynamic Service load-balancing rules into the host `iptables`.**

### 7. Peak 1 — how [[CNI]] actually carries Pod traffic across nodes

#### Why it's a problem

Pod IPs (e.g. `10.244.x.x`) are a virtual range internal to k8s. **The physical network between nodes doesn't know that range** — it only knows the nodes' real IPs (`192.168.1.x`). Bridging this gap is the CNI's job. Two approaches.

#### Approach A: Overlay (VXLAN) — "put it in an envelope"

Representative: [[Flannel]] (default), [[Calico]] VXLAN mode.

**Wrap** the Pod packet that the physical network doesn't understand inside a packet the physical network does understand.

```
Inner packet:  [src=10.244.1.5  dst=10.244.2.7  payload]
                                ↓ encapsulate
Outer packet:  [src=192.168.1.10 dst=192.168.1.20  UDP:8472  VXLAN-header | inner ]
                                ↓ physical net routes this happily
                  Destination node → unwrap → deliver inner to Pod B
```

Flow:
1. Node1's `flannel.1` (VXLAN device) intercepts, looks up "10.244.2.7 lives on Node2" (FDB)
2. Wraps it in a UDP envelope and sends
3. Node2's `flannel.1` unwraps → hands off to Pod B

| Pros | Cons |
|---|---|
| Works even on dumb physical networks | Encapsulation CPU overhead |
| Crosses cloud [[VPC]] boundaries | MTU shrinks (~50 bytes) |
| | Harder to debug (`tcpdump` sees the envelope) |

#### Approach B: Routing (L3/BGP) — "post signs along the road"

Representative: [[Calico]] BGP, [[Cilium]], [[Flannel]] `host-gw`.

No envelope — **each node advertises its own Pod range**, and the routing table delivers packets directly.

```
Node1 routing table:
  10.244.2.0/24  via  192.168.1.20  (Node2)
  10.244.3.0/24  via  192.168.1.30  (Node3)

→ Pod packet stays raw: [src=10.244.1.5 dst=10.244.2.7]
→ Network reads routing table, forwards to Node2
```

Routes sync between nodes via [[BGP]] (Calico uses the [[BIRD]] daemon). **Packets travel unwrapped.**

| Pros | Cons |
|---|---|
| Near-native speed | Physical network must be capable (L3 reachability / BGP support) |
| No MTU shrinkage | Trickier if nodes sit in different subnets |
| Easy to debug | |

One line: **Both solve "physical network doesn't know Pod IPs." Overlay wraps and reroutes; routing teaches the network the way.**

### 8. Peak 2 — anatomy of [[kube-proxy]]'s `iptables` chains

#### Overall layering

```
Packet enters (PREROUTING/OUTPUT)
        ↓
┌───────────────────┐
│  KUBE-SERVICES    │  ← classify which Service (matches the VIP)
└────────┬──────────┘
         ↓
┌───────────────────┐
│  KUBE-SVC-XXX     │  ← load balance (probability split)
└────────┬──────────┘
         ↓
┌───────────────────┐
│  KUBE-SEP-YYY     │  ← actual DNAT (VIP → real Pod IP)
└───────────────────┘
```

`SVC` = Service, `SEP` = Service EndPoint (one Pod).

#### ① `KUBE-SERVICES` — traffic sorter

One row per Service VIP.

```
-d 10.96.0.10/32 -p tcp --dport 3000 -j KUBE-SVC-ABC
-d 10.96.0.11/32 -p tcp --dport 80   -j KUBE-SVC-DEF
...
```

Only decides which Service row this packet belongs to.

#### ② `KUBE-SVC-XXX` — the load balancer (probability split)

Uses the `statistic` module.

```
-m statistic --mode random --probability 0.33333  -j KUBE-SEP-001
-m statistic --mode random --probability 0.50000  -j KUBE-SEP-002
                                                  -j KUBE-SEP-003   (catch-all)
```

> [!tip] Why probabilities are 0.33 → 0.5 → 1.0
> Sequential filtering. First row catches 1/3. Of the remaining 2/3, half (= 1/3 overall) hits the second. The final 1/3 falls through to the third. **Perfect three-way split.**

#### ③ `KUBE-SEP-YYY` — the actual DNAT

One SEP = one Pod.

```
(a) -s 10.244.1.5/32 -j KUBE-MARK-MASQ            ← mark for hairpin SNAT
(b) -p tcp -j DNAT --to-destination 10.244.1.5:3000   ← VIP → real Pod IP
```

Line (b) **is** DNAT — the rewrite from virtual VIP to real Pod IP.

#### Two missing pieces

**(1) `KUBE-MARK-MASQ` / `KUBE-POSTROUTING` (SNAT)**

If you only change the destination, replies can flow to the wrong place (hairpin traffic, packets originating outside the cluster). Marked packets get `MASQUERADE` applied in `KUBE-POSTROUTING` to also rewrite the source.

**(2) [[conntrack]] (connection tracking)**

The kernel remembers the outbound DNAT, so **reply packets are automatically reverse-translated** (Pod IP back to VIP source). DNAT only runs on the first packet of a flow — the rest ride conntrack — which makes it fast and keeps sessions consistent.

> [!note] Where the two peaks meet
> The `KUBE-SEP` step picks "destination = some Pod IP on some node." Actual delivery to that Pod is Peak 1's job ([[CNI]] overlay/routing). **iptables decides which Pod and rewrites the address; CNI carries the packet there.**

### 9. Probes — deciding Pod liveness and readiness

#### Three different questions

| Probe | Question | On failure |
|---|---|---|
| **liveness** | "Are you alive?" | Kill and restart the container |
| **readiness** | "Ready to accept traffic?" | Remove from Service endpoints (no restart) |
| **startup** | "Done booting yet?" | Hold off the other two until it passes |

Distinguishing **liveness ≠ readiness** is the key.

#### readiness probe — the deferred topic

Earlier we said "when a new Pod comes up, we have to confirm it's truly ready before adding it to the endpoints" — that's readiness. A container being "up" doesn't mean it can serve traffic; DB pool warm-up, cache priming, config load all take time.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  periodSeconds: 5
  failureThreshold: 3
```

Connection flow:
```
readiness passes
   → kubelet reports Ready to API Server
   → EndpointSlice controller adds Pod IP to the endpoint list
   → kube-proxy adds an iptables row to KUBE-SVC
   → traffic starts flowing

readiness fails
   → removed from endpoint list
   → iptables row removed
   → no traffic (container is NOT killed)
```

**Readiness is the on/off switch for endpoint membership** — and the reason rolling updates achieve zero downtime (new Pod doesn't get traffic until readiness passes).

#### liveness probe

Detects "have you turned into a zombie?" (deadlock, etc.). Failure → [[kubelet]] kills and restarts the container (repeated failure → `CrashLoopBackOff`).

#### startup probe

If your app takes 2 minutes to boot, liveness alone would loop-restart it forever during boot. The startup probe **disables liveness/readiness until it passes** (max wait = `failureThreshold × periodSeconds`). Then the others take over.

#### Four probe types

- `httpGet` — HTTP status 200–399 = pass
- `tcpSocket` — pass if the port accepts a connection
- `exec` — command exit code 0 = pass
- `grpc` — gRPC health check

> [!warning] Common production mistakes
> 1. **Putting dependency checks in liveness** → when the DB blips, all your healthy containers restart in a cascade. liveness = "am I a zombie?"; **dependencies belong in readiness**.
> 2. **No probes at all** → traffic hits not-yet-ready Pods.
> 3. **Too-tight initialDelay/threshold** → healthy Pods get restart-looped.

### 10. [[eBPF]] / [[Cilium]] — skipping `iptables` entirely

#### Why ditch iptables

1. **Linear O(n)** scan of `KUBE-SERVICES`. Thousands of Services → tens of thousands of rules
2. **Update cost** — changing one Pod rewrites chains
3. **User/kernel boundary, conntrack overhead**

Fine for small clusters; becomes a bottleneck at thousands of Services.

#### What [[eBPF]] is

"Small, sandboxed programs that run inside the kernel" — **"JavaScript for the kernel."** No recompile, no reboot — verified programs are attached to kernel hooks.

```
┌──────────────── Linux Kernel ────────────���───┐
│                                              │
│   [verifier] ← validates eBPF code            │
│                 (no loops, memory-safe)       │
│        ↓                                     │
│   [hook]  XDP, tc, socket, kprobe...         │
│        ↓                                     │
│   [eBPF program] runs inside kernel (fast)   │
│        ↕                                     │
│   [eBPF map]  hash table  ← "VIP → Pod"      │
└──────────────────────────────────────────────┘
```

Properties:
- Runs in kernel (fast)
- Verifier enforces safety
- eBPF maps (hash tables) store "VIP → Pod" → **O(1) lookups**
- Attaches at packet-path hooks (XDP/tc/socket)

#### What [[Cilium]] does

Replaces `kube-proxy` wholesale.

| Old [[iptables]] | [[Cilium]]/[[eBPF]] |
|---|---|
| O(n) linear scan | O(1) hash lookup |
| Probability split → DNAT | Map gives the target directly |
| Slower as rules grow | Latency independent of scale |
| Chain rewrites | Update one map entry |

Crucially, **socket-level load balancing**: for internal (east-west) calls, Cilium fixes the destination at the `connect()` syscall — **no per-packet DNAT needed**. conntrack overhead drops too. Near-zero overhead.

#### Benefits

- **O(1) hash map** → scale-invariant
- **One map row** to update → instant change
- **Socket-level LB** (one decision per connection)
- **Identity-based (label-based) security** — policy survives Pod IP churn, extends to L7
- **[[Hubble]] observability** — eBPF-powered traffic introspection

Trade-off: needs a **relatively new kernel**.

> [!note] How it relates to Peak 1
> [[Cilium]] is **the all-in-one** — [[CNI]] role + `kube-proxy` role + [[NetworkPolicy]] enforcement, all via eBPF.

One sentence: **`iptables` is the old approach that linearly walks rules and gets slow at scale. eBPF runs an O(1) hash map inside the kernel and load-balances at the socket layer; Cilium uses this to replace kube-proxy, CNI, and security policy in one shot.**

## Diagram

[[canvas/Kubernetes_개념과_네트워킹_아키텍처.canvas|Concept map]]
