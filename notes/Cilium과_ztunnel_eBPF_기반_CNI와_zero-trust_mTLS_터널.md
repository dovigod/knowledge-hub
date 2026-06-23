---
id: 019ef3b2-7655-761c-83ca-2cd93bcda896
title: 'Cilium과 ztunnel: eBPF 기반 CNI와 zero-trust mTLS 터널'
topics:
  - cilium
  - ztunnel
  - cni
  - ebpf
  - mtls
  - kubernetes-networking
tags:
  - networking
  - kubernetes
  - cilium
  - ztunnel
  - mtls
  - ebpf
  - security
  - education
sources:
  - 019ed89a-6333-736b-a257-45a8581f1a66
created_at: '2026-06-23T08:56:59.732Z'
updated_at: '2026-06-23T08:56:59.732Z'
---
> [!note] TL;DR
> **[[Cilium]]**은 [[eBPF]]를 커널에 심어 패킷을 처리하는 [[CNI]] 플러그인이고, **[[ztunnel]]**은 사이드카 없이 노드당 하나만 도는 [[zero-trust]] 프록시로 Pod 간 트래픽을 [[mTLS]]로 자동 암호화한다. 이 클러스터는 `encryption.type: ztunnel`로 두 기능을 결합해 쓰며, 그 [[mTLS]] 신뢰의 뿌리(CA)인 `cilium-ztunnel-secrets`가 [[data plane]]의 필수 부트스트랩 — 시크릿이 없으면 cilium-agent가 못 떠서 [[CNI]] 자체가 없고, 노드가 NotReady로 떨어져 클러스터가 통째로 cascade된다.

## 핵심 그림 한 장

```
Pod A (node1)                              Pod B (node2)
   │ 평문                                       ▲ 평문
   ▼                                            │
[Cilium eBPF: 트래픽 redirect]           [Cilium eBPF]
   │                                            ▲
   ▼                                            │
ztunnel(node1) ──── mTLS HBONE 터널 ────► ztunnel(node2)
   암호화                  네트워크              복호화
        ▲                                  ▲
        └── cilium-ztunnel-secrets의 CA ───┘
            (ca-root.crt / ca-private.key)
            발급된 인증서로 SPIFFE ID 검증
            (O = cluster.local)
```

## 이 repo에 ztunnel이 두 종류 있다 — 먼저 정리

| | **Cilium ztunnel 암호화** (지금 컨텍스트) | **Istio ztunnel** |
|---|---|---|
| 위치 | `bootstrap-v2-cilium.yaml` `encryption.type: ztunnel` | `charts/bootstrap/templates/08_istio-ztunnel.yaml` |
| 네임스페이스 | `kube-system` | `istio-system` |
| 시크릿 | `cilium-ztunnel-secrets` | Istio CA (istiod) |
| 활성화 | 사용 중 | `istio.enabled: false` (꺼짐) |

> [!warning] 헷갈리지 말 것
> `cilium-ztunnel-secrets`는 **Cilium이 자체적으로 임베드한 ztunnel**용이다. [[Istio]]를 별도로 깐 게 아니라, [[Cilium]]이 암호화 백엔드로 [[ztunnel]] 메커니즘을 채택한 형태다. 그래서 시크릿이 `istio-system`이 아니라 `kube-system`(=`cilium-agent`와 같은 곳)에 산다.

## CNI가 뭐야

**[[CNI]] = Container Network Interface.** 한마디로 "쿠버네티스가 Pod에 네트워크를 붙이는 표준 규격(플러그인 인터페이스)".

쿠버네티스는 **네트워킹을 직접 구현하지 않는다.** 요구사항만 정한다:

- 모든 Pod는 고유 IP를 가진다
- 모든 Pod는 NAT 없이 서로 통신 가능
- 노드도 Pod와 통신 가능

"이걸 어떻게 구현할지는 너희가 알아서 해" → 그 **플러그인을 끼우는 표준 구멍이 [[CNI]]**.

```
쿠버네티스 (kubelet)
      │  "Pod 떴다. 네트워크 좀 붙여줘"
      ▼
 [ CNI 인터페이스 ]  ← 표준 규격(구멍)
      │
      ▼
 CNI 플러그인  ← 실제 구현체 (Cilium / Calico / Flannel ...)
```

### 동작 흐름

1. Pod 생성 → kubelet이 **network namespace** 만듦
2. kubelet이 **CNI 플러그인을 `ADD` 명령으로 호출**
3. 플러그인이:
   - **veth(가상 케이블)** 만들어 Pod ↔ 노드 연결
   - Pod에 **IP 할당** (IPAM)
   - 라우팅·정책 설정
4. Pod 삭제 시 `DEL` 호출 → 정리

즉 [[CNI]]는 "Pod가 뜰 때마다 네트워크를 꽂아주고, 죽을 때 뽑아주는" 표준화된 약속.

## CNI로서 Cilium의 기본 원리 (eBPF)

[[Cilium]]은 [[iptables]] 기반 [[kube-proxy]]를 대체하는 **[[eBPF]] 기반 [[CNI]]**다.

- Pod가 뜨면 `cilium-agent`가 veth를 만들고, **커널 [[eBPF]] 프로그램**을 네트워크 훅(tc/XDP)에 붙인다.
- 라우팅·정책·로드밸런싱을 [[iptables]] 룰 체인이 아니라 **[[eBPF]] 맵 lookup**으로 처리한다.
- "**identity**" 개념: IP가 아니라 **Pod label 기반 보안 ID**로 정책 적용. IP가 바뀌어도 정책은 유지.

### iptables vs eBPF — 성능 차이의 본질

```
일반 CNI(iptables):  패킷 → 룰1 → 룰2 → ... → 룰5000   (선형 탐색, 룰 많으면 느려짐)
Cilium(eBPF):        패킷 → eBPF 맵 해시 조회 1번        (즉시 결정)
```

> [!tip] 비유
> [[iptables]]가 **"규정집을 첫 장부터 한 줄씩 읽어 내려가며 검문"**이라면, [[eBPF]]는 **"입구에서 바코드 한 번 찍으면 즉시 통과/차단 판정"**. 규정이 5000개로 늘어도 속도가 안 떨어진다(해시 조회라서).

> [!warning] Cilium = 데이터플레인 그 자체
> `cilium-agent`가 못 뜨면 노드 네트워킹 전체가 죽는다(=이번에 본 cascade). [[Cilium]]은 [[kube-proxy]] 같은 보조가 아니라 **패킷이 지나가는 길 자체**를 만드는 주체다.

## mTLS는 뭐야

**[[mTLS]] = mutual TLS** (상호 TLS). "TLS인데 양쪽 다 인증서로 신원을 증명하는 것".

### 일반 TLS vs mTLS

평소 HTTPS(브라우저 ↔ 웹서버)는 **한쪽만** 인증한다.

```
일반 TLS (단방향)
브라우저 ──"넌 진짜 naver.com이야?"──► 서버
         ◄── 서버 인증서 제출 ──        (서버만 증명)
브라우저는? 신원 증명 안 함 (그냥 익명 손님)
```

[[mTLS]]는 **양쪽 다** 인증서를 제출해서 서로를 검증한다.

```
mTLS (양방향)
A ──"넌 누구야?"──► B   B가 인증서 제출
A ◄──"너는 누구야?"── B   A도 인증서 제출
둘 다 상대 인증서를 CA로 검증한 뒤에야 통신 시작
```

### 무엇을 보장하나

| 보장 | 설명 |
|---|---|
| **암호화** | 중간에서 패킷 까봐도 내용 못 봄 (일반 TLS도 됨) |
| **상호 신원 인증** | "상대가 진짜 그 워크로드가 맞는가"를 **양쪽 다** 확인 (mTLS만) |
| **위변조 방지** | 가짜 Pod가 끼어들어 트래픽 가로채기 불가 |

### 왜 클러스터 내부 통신에 쓰나 ([[zero-trust]])

전통적 보안은 "방화벽 안쪽 = 안전"이라 믿었지만, [[zero-trust]]는 **내부 트래픽도 못 믿는다**가 전제.

- 일반 TLS는 클라이언트가 익명 → 악성 Pod가 다른 서비스로 위장 접속 가능.
- [[mTLS]]는 **양쪽 다 인증서가 있어야** 연결되니까, 인증서 없는(=신원 없는) Pod는 통신 자체가 안 됨.

### 우리 클러스터 맥락

```
ztunnel(node1) ◄── 서로 인증서 제출 & 검증 ──► ztunnel(node2)
       │                                          │
   cilium-ztunnel-secrets의 CA로 발급받은 인증서
       │                                          │
   SPIFFE ID 검증: spiffe://cluster.local/...
```

- 각 [[ztunnel]]은 `cilium-ztunnel-secrets`의 **CA로 발급받은 인증서**를 들고 있음.
- 노드 간 연결 시 서로 인증서 교환 → "너 진짜 같은 `cluster.local` 신뢰 도메인 소속 맞아?" 확인.
- 그 **CA(`ca-root.crt`/`ca-private.key`)가 mTLS 전체의 신뢰 뿌리** → 없으면 인증서 발급 자체가 안 되고 mTLS가 성립 못 함.

## ztunnel이 뭐야 — 사이드카 없는 노드 프록시

[[ztunnel]] = "**zero-trust tunnel**". 원래 [[Istio Ambient Mesh]]의 L4 데이터플레인이다.

### 먼저 "프록시가 왜 필요한가" — 경비원 비유

Pod끼리 그냥 통신하면 암호화·신원확인·접근제어를 각 앱이 직접 코드로 짜야 한다. 그래서 각 통신 앞에 **"경비원"**을 세운다. 이 경비원이 대신 [[mTLS]] 암호화·신원확인·출입통제를 해준다. **그 경비원이 프록시**.

### 옛날 방식 (사이드카) — "방마다 경비원 1명"

전통적 [[service mesh]]([[Istio]] 기본 모드)는 **Pod마다 Envoy 프록시를 하나씩** 붙인다.

```
사이드카 모델 = 아파트 모든 집 현관마다 경비원 배치

[Pod A][경비원]   [Pod B][경비원]   [Pod C][경비원]
[Pod D][경비원]   [Pod E][경비원]   [Pod F][경비원]
```

문제:
- **경비원 100명** (Pod 100개면 프록시 100개) → 메모리·CPU 낭비 엄청남
- 집(앱)을 새로 단장(배포)할 때마다 **경비원도 같이 재배치** → 앱 업데이트하면 프록시도 재시작
- 작은 집(가벼운 Pod)에도 똑같이 무거운 경비원이 붙음 → 배보다 배꼽

### 새 방식 (ztunnel) — "건물 1층 로비에 경비실 하나"

[[ztunnel]]은 **노드당 프록시 하나(DaemonSet)**만 둔다.

```
ztunnel 모델 = 아파트 1동에 공동 경비실 하나

       ┌─── 노드(아파트 1동) ───┐
       │ [Pod A] [Pod B] [Pod C] │
       │ [Pod D] [Pod E] [Pod F] │
       │                         │
       │   [ztunnel 경비실 1개]  │ ← 이 동의 모든 출입을 관리
       └─────────────────────────┘
```

이 동의 모든 Pod 트래픽을 **공동 경비실 하나가 담당**.

### 비교 — 3가지 이득

| | 사이드카 (방마다 경비원) | ztunnel (동마다 경비실) |
|---|---|---|
| **리소스** | Pod 100개 = 프록시 100개 (무거움) | 노드 5개 = 프록시 5개 (가벼움) |
| **배포 영향** | 앱 배포 시 프록시도 재시작 | 앱 따로, 프록시 따로 |
| **운영** | 프록시 100개 관리·버전업 | 노드당 하나만 관리 |

특히 두 번째가 크다. 사이드카는 **앱과 보안이 한 몸** → 앱 건드릴 때마다 보안 레이어도 흔들림. [[ztunnel]]은 **보안 인프라를 앱에서 떼어내서** 노드 레벨로 내린다.

### ztunnel이 실제로 하는 일

- **mTLS 암호화/복호화** (L4, **HBONE 터널** = HTTP/2 CONNECT 위에 mTLS)
- **워크로드 신원(identity) 부여** — [[SPIFFE]] 형식 `spiffe://cluster.local/ns/<ns>/sa/<sa>`
- L7은 안 함 (그건 [[Istio]]의 **waypoint proxy** 몫). [[ztunnel]]은 **순수 L4 + mTLS**만.

> [!note] 트레이드오프
> 공동 경비실(ztunnel)은 **누가 드나드는지(L4)만** 본다. 세밀한 L7 검문은 못 한다. 우리는 [[Cilium]] 암호화용으로 그 L4 mTLS 기능만 쓰는 거라 그걸로 충분하다.

## Cilium + ztunnel mTLS가 실제로 도는 방식

1. **트래픽 가로채기**: [[Cilium]]의 [[eBPF]]가 Pod에서 나가는 패킷을 노드의 [[ztunnel]]로 redirect (사이드카 주입 없이 커널 레벨에서).
2. **신원 확인 + mTLS**: [[ztunnel]]이 `cilium-ztunnel-secrets`의 CA로 발급받은 인증서로 상대 노드 [[ztunnel]]과 [[mTLS]] 핸드셰이크. 서로의 [[SPIFFE]] ID(`O = cluster.local`)를 검증.
3. **터널링**: 암호화된 트래픽이 노드 간 네트워크를 건너간다. [[IPsec]]/[[WireGuard]] 대신 [[ztunnel]]의 mTLS 터널이 그 역할.
4. **복호화 + 전달**: 상대 노드 [[ztunnel]]이 복호화 → [[eBPF]]가 목적지 Pod로 전달.

## 왜 시크릿이 "옵션"이 아니라 "부트스트랩 전제"인가

```
cilium-ztunnel-secrets:
  ├─ ca-root.crt / ca-private.key          ← 워크로드 인증서를 발급하는 루트 CA
  └─ bootstrap-root.crt / bootstrap-private.key  ← ztunnel 기동용 부트스트랩 인증서
```

- `encryption.type: ztunnel` → [[ztunnel]]이 **데이터플레인의 필수 구성요소**.
- `cilium-agent`는 [[ztunnel]] 인증서 볼륨을 `optional: false`로 마운트 → 시크릿 없으면 agent Pod가 `ContainerCreating`에서 멈춤.
- CA(`ca-root.crt`/`ca-private.key`)는 **워크로드 인증서 발급용 루트**, bootstrap 인증서는 [[ztunnel]] **기동 시 자기 자신을 증명**하는 초기 신뢰 앵커.
- agent 안 뜸 → [[CNI]] 없음 → 노드 `NotReady` → 클러스터 cascade.

> [!warning] 부트스트랩 chicken-and-egg
> 35 클러스터 신규 구성 시 `scripts/ztunnel_generate_secret.sh`로 **시크릿을 가장 먼저** 만들어줘야 한다. 데이터플레인을 만들려면 데이터플레인의 신뢰 뿌리가 먼저 있어야 한다는 부트스트랩 전제조건.

## Flannel / Calico / Cilium CNI 비교

핵심은 두 가지 질문:

1. **다른 노드의 Pod에 패킷을 어떻게 전달하나?** (오버레이 vs 라우팅)
2. **정책·서비스 처리를 무엇으로 하나?** (iptables vs eBPF)

### 1. [[Flannel]] — "택배 상자에 넣어 보내기" (VXLAN 오버레이)

가장 단순한 [[CNI]]. 노드 간 통신을 **오버레이**로.

**문제**: 노드1 Pod(`10.244.1.5`) → 노드2 Pod(`10.244.2.8`). 물리 네트워크는 Pod IP를 모름. 노드 IP(`192.168.x.x`)만 안다.

**VXLAN 해결법** — "상자 안에 상자":

```
원본 패킷:        [출발 Pod IP → 목적 Pod IP][데이터]
                          │ VXLAN이 통째로 감쌈
                          ▼
캡슐화된 패킷:   [노드1 IP → 노드2 IP][ {원본 패킷} ]
                  └─ 물리망이 아는 주소 ─┘
```

> [!tip] 비유
> 회사 내부 사서함 주소(Pod IP)로는 우체국이 배달 못 하니까, **회사 건물 주소(노드 IP)가 적힌 큰 택배 상자에 원래 편지를 넣어** 보낸다. 받는 회사가 상자를 뜯어서 내부 사서함으로 전달.

- **장점**: 물리 네트워크 무관, 어디서나 동작
- **단점**: 매번 포장/개봉 → 오버헤드, 느림
- **정책**: 없음

### 2. [[Calico]] — "진짜 주소로 직접 배달" (BGP 라우팅 + iptables 정책)

**(a) 전달 방식** — 오버레이 안 씀, 라우팅.
[[Calico]]는 상자에 안 넣고 **라우터에게 Pod IP로 가는 길을 직접 알려준다.** 이때 쓰는 게 **[[BGP]]**.

```
Flannel: 포장해서 보냄         (오버레이, 느림)
Calico:  라우터가 Pod IP 길을 이미 알아서 그냥 직접 보냄  (네이티브 라우팅, 빠름)
```

> [!tip] 비유
> [[Flannel]]이 "택배 상자에 포장"이라면, [[Calico]]는 **우체국(라우터)에 모든 사서함 주소를 미리 등록**해놔서 포장 없이 직접 배달. 단, 우체국(물리망)이 협조해줘야 함([[BGP]] 지원).

**(b) 정책** — [[iptables]] 룰로 NetworkPolicy 구현.

```
패킷 도착 → iptables 룰 체인을 위에서부터 검사
  룰1: A→B 허용?  아니오 ↓
  룰2: C→D 차단?  아니오 ↓
  ...
  룰5000: ...      (룰 많아지면 선형 탐색 → 느려짐)
```

- **장점**: 정책 강력·성숙, 오버레이 없어 빠름
- **단점**: 정책 많으면 [[iptables]] 길어져 성능 저하, [[BGP]]가 물리망에 의존

### 3. [[Cilium]] — "커널에 직접 프로그램 심기" ([[eBPF]])

앞 둘은 리눅스가 원래 주는 도구(VXLAN, iptables) 조합이라면, [[Cilium]]은 **커널 안에 작은 프로그램([[eBPF]])을 직접 심어서** 패킷 처리.

```
Calico(iptables):  패킷 → 룰1 → 룰2 → ... → 룰5000   (선형)
Cilium(eBPF):      패킷 → eBPF 맵 해시 조회 1번        (즉시)
```

추가로 하는 것들:

| 기능 | 설명 |
|---|---|
| **iptables 대체** | [[kube-proxy]](서비스 LB)까지 eBPF로 대체 |
| **고성능** | 룰 수와 무관하게 일정한 속도 |
| **L7 정책** | HTTP 경로·메서드까지 제어 (`GET /api` 허용, `DELETE` 차단) |
| **identity 기반** | IP 아니라 **Pod label**로 정책 — IP 바뀌어도 정책 유지 |
| **암호화** | [[IPsec]]/[[WireGuard]]/**[[ztunnel]] mTLS** 내장 ← 우리가 쓰는 것 |

### 한눈에 비교

```
                Flannel          Calico            Cilium
노드 간 전달    택배 포장        직접 배달         직접 배달
                (VXLAN)          (BGP 라우팅)      (eBPF 라우팅)
정책 엔진       없음             iptables          eBPF
정책 수준       —                L3/L4            L3/L4/L7
속도            보통             빠름              가장 빠름(룰↑무관)
암호화          ✕                제한적            IPsec/WG/ztunnel
복잡도          낮음             중간              높음
```

**진화의 흐름:**
- [[Flannel]] = "되게만 하자"
- [[Calico]] = "정책도 제대로, 오버레이 빼서 빠르게"
- [[Cilium]] = "[[iptables]] 한계를 [[eBPF]]로 넘자, L7·암호화까지"

> [!note] 우리 클러스터가 Cilium인 이유
> 다른 [[CNI]]였으면 [[mTLS]]를 별도 메시([[Istio]] 등)로 또 깔아야 했다. [[Cilium]]은 **[[CNI]] 안에 암호화까지 통합**되어 있어서 [[ztunnel]] 방식으로 노드 간 트래픽을 자동 암호화한다.

## 데이터플레인은 뭐야

[[data plane]]은 **"실제로 패킷(데이터)이 지나다니는 길"**. 짝이 되는 개념인 [[control plane]]과 같이 봐야 이해된다.

### 두 개의 플레인 — 핵심 구분

```
컨트롤플레인 (Control Plane)  =  "결정하는 뇌"
   → 어떻게 라우팅할지, 누가 통신 가능한지 규칙을 정함

데이터플레인 (Data Plane)     =  "실제로 나르는 손발"
   → 그 규칙대로 실제 패킷을 전달함
```

> [!tip] 우체국 비유
> - **[[control plane]]** = 관제실. "이 지역은 3번 트럭이, 저 지역은 5번 트럭이 나른다"는 **경로표(라우팅 테이블)**를 만듦. 편지를 직접 만지진 않음.
> - **[[data plane]]** = 실제로 편지를 싣고 달리는 **트럭들.** 경로표를 보고 목적지로 나름.
>
> 관제실이 잠깐 멈춰도 트럭들은 기존 경로표로 계속 배달 가능하지만, **트럭이 멈추면 배달 자체가 멈춘다.**

### 네트워크에서

| | 컨트롤플레인 | 데이터플레인 |
|---|---|---|
| 역할 | 규칙·경로 **결정** | 패킷 **전달** |
| 빈도 | 가끔 (정책 바뀔 때) | 항상 (패킷마다) |
| 속도 요구 | 덜 중요 | **매우 중요** (모든 패킷이 지나감) |
| 예시 | 라우팅 프로토콜, 정책 계산 | 패킷 포워딩, 암호화/복호화 |

### Cilium에서

```
컨트롤플레인:  cilium-operator, 정책 계산, identity 관리
              "Pod label X는 Pod Y와 통신 가능" 같은 규칙을 계산해서
              eBPF 맵에 써넣음

데이터플레인:  커널의 eBPF 프로그램들
              실제 패킷이 지나갈 때 그 맵을 조회해서 즉시 전달/차단
```

### 우리 클러스터 — 왜 "[[Cilium]] = 데이터플레인 그 자체"인가

```
일반 환경:
  네트워킹 = 컨트롤플레인(쿠버네티스) + 데이터플레인(여러 도구가 분담)
  → 일부 죽어도 나머지로 버팀

Cilium 환경:
  데이터플레인을 Cilium의 eBPF가 통째로 담당
  → 패킷이 지나가는 길 자체가 Cilium
```

`encryption.type: ztunnel`이라 **[[ztunnel]]의 [[mTLS]] 암호화도 [[data plane]]의 일부**:

```
데이터플레인 = eBPF 포워딩 + ztunnel mTLS 암호화
                                    │
                  cilium-ztunnel-secrets가 없으면
                  cilium-agent가 못 뜸
                                    │
                  데이터플레인 전체가 안 만들어짐
                  = 패킷 길 자체가 없음
                  = 노드 NotReady → 클러스터 cascade
```

## 부록: L4 vs L7 정책

| 트래픽 | L7 가능? | 왜 |
|---|---|---|
| HTTP | ✅ | 표준 텍스트, 파싱 쉬움 |
| gRPC | ✅ | HTTP/2 기반 |
| Kafka, DNS | ✅ | Cilium에 전용 파서 있음 |
| PostgreSQL/MySQL/Redis | ❌ → L4 | 전용 바이너리, 파서 없음 |
| TLS 암호화 트래픽 | ❌ → L4 | 내용이 암호문이라 못 읽음 |
| 일반 TCP/UDP 스트림 | ❌ → L4 | 구조 자체가 없음 |

핵심 원리:
> **L7 정책은 "프록시가 그 프로토콜을 평문으로 읽고 이해할 수 있을 때만" 가능하다.**

## 한 줄 요약

> [[Cilium]]([[eBPF]] [[CNI]])이 패킷을 노드별 [[ztunnel]] 프록시로 redirect하고, [[ztunnel]]이 `cilium-ztunnel-secrets` CA로 발급한 인증서로 노드 간 트래픽을 [[mTLS]] 자동 암호화한다. 사이드카가 없으니 가볍고, CA가 [[data plane]]의 신뢰 뿌리라서 없으면 네트워킹 전체가 안 뜬다.

## 다이어그램

[[canvas/Cilium과_ztunnel_eBPF_기반_CNI와_zero-trust_mTLS_터널.canvas|개념도]]

---

## English

> [!note] TL;DR
> **[[Cilium]]** is a [[CNI]] plugin that processes packets via [[eBPF]] programs loaded into the kernel, and **[[ztunnel]]** is a [[zero-trust]] proxy that runs once per node (no sidecars) and transparently encrypts pod-to-pod traffic with [[mTLS]]. This cluster combines both via `encryption.type: ztunnel`, so the [[mTLS]] root of trust — the `cilium-ztunnel-secrets` CA — is a mandatory [[data plane]] bootstrap: without it, `cilium-agent` can't start, there's no [[CNI]], nodes go NotReady, and the cluster cascades.

### The one diagram

```
Pod A (node1)                              Pod B (node2)
   │ plaintext                                  ▲ plaintext
   ▼                                            │
[Cilium eBPF: redirect traffic]          [Cilium eBPF]
   │                                            ▲
   ▼                                            │
ztunnel(node1) ──── mTLS HBONE tunnel ────► ztunnel(node2)
   encrypts               network               decrypts
        ▲                                  ▲
        └── CA from cilium-ztunnel-secrets ┘
            (ca-root.crt / ca-private.key)
            issues certs, SPIFFE ID verified
            (O = cluster.local)
```

### Two "ztunnels" in this repo — disambiguation first

| | **Cilium ztunnel encryption** (this context) | **Istio ztunnel** |
|---|---|---|
| Location | `bootstrap-v2-cilium.yaml` `encryption.type: ztunnel` | `charts/bootstrap/templates/08_istio-ztunnel.yaml` |
| Namespace | `kube-system` | `istio-system` |
| Secret | `cilium-ztunnel-secrets` | Istio CA (istiod) |
| Active | in use | `istio.enabled: false` (off) |

> [!warning] Don't confuse them
> `cilium-ztunnel-secrets` belongs to the **ztunnel that [[Cilium]] embeds natively**. This isn't a separate [[Istio]] install — [[Cilium]] adopted the [[ztunnel]] mechanism as its encryption backend. That's why the secret sits in `kube-system` (next to `cilium-agent`), not `istio-system`.

### What is a CNI

**[[CNI]] = Container Network Interface.** The standard plugin contract Kubernetes uses to attach networking to pods.

Kubernetes itself **does not implement networking.** It only sets requirements:

- Every pod has a unique IP
- Every pod can talk to every other pod without NAT
- Nodes can talk to pods

"How you implement that is up to you" → that **plugin socket is [[CNI]]**.

```
Kubernetes (kubelet)
      │  "new pod — wire up networking"
      ▼
 [ CNI interface ]  ← standard socket
      │
      ▼
 CNI plugin  ← actual implementation (Cilium / Calico / Flannel ...)
```

#### Lifecycle

1. Pod created → kubelet makes a **network namespace**
2. kubelet calls **CNI plugin with `ADD`**
3. Plugin:
   - Creates **veth** (virtual cable) between pod and node
   - Allocates pod **IP** (IPAM)
   - Sets up routing/policy
4. Pod deleted → `DEL` → teardown

[[CNI]] is the standardized contract: "plug networking in when a pod is born, unplug it when it dies."

### Cilium as a CNI — the eBPF foundation

[[Cilium]] is an **[[eBPF]]-based [[CNI]]** that replaces [[iptables]]-based [[kube-proxy]].

- When a pod starts, `cilium-agent` creates the veth and **attaches kernel [[eBPF]] programs** to network hooks (tc/XDP).
- Routing, policy, and load balancing happen via **[[eBPF]] map lookups**, not [[iptables]] rule chains.
- "**identity**" concept: policy attaches to **pod labels**, not IPs. Pod IP rotates, policy stays valid.

#### iptables vs eBPF — why it matters

```
Traditional CNI(iptables):  pkt → rule1 → rule2 → ... → rule5000   (linear scan)
Cilium(eBPF):               pkt → eBPF map hash lookup, once       (instant)
```

> [!tip] Analogy
> [[iptables]] is **"reading a rulebook line by line at the gate"**. [[eBPF]] is **"swipe a barcode at the door — instant pass/block."** Scales to 5000 rules without slowdown because it's a hash lookup.

> [!warning] Cilium IS the data plane
> If `cilium-agent` can't start, the node has no networking — that's the cascade we saw. [[Cilium]] is not a helper alongside [[kube-proxy]] — it owns the path packets travel.

### What is mTLS

**[[mTLS]] = mutual TLS.** TLS where both sides prove identity with certificates.

#### Regular TLS vs mTLS

Normal HTTPS (browser ↔ web server) authenticates **one side**:

```
Regular TLS (one-way)
Browser ──"are you really naver.com?"──► Server
        ◄── server cert ───              (only server proves)
Browser is anonymous
```

[[mTLS]] makes **both sides** present certs:

```
mTLS (two-way)
A ──"who are you?"──► B   B sends cert
A ◄──"who are you?"── B   A sends cert
Both verify against CA before any traffic flows
```

#### What it guarantees

| Property | What |
|---|---|
| **Encryption** | Wire-tap shows ciphertext (regular TLS too) |
| **Mutual identity** | "Is the peer really that workload?" verified **both ways** (mTLS only) |
| **Tamper-proof** | No impostor pod can splice into the connection |

#### Why inside-the-cluster ([[zero-trust]])

Old security trusted "inside the firewall = safe." [[zero-trust]] assumes **east-west traffic is hostile too**.

- Regular TLS → client is anonymous → a rogue pod can impersonate a service.
- [[mTLS]] → both sides need certs → a pod with no cert (no identity) literally cannot connect.

#### In this cluster

```
ztunnel(node1) ◄── exchange & verify certs ──► ztunnel(node2)
       │                                          │
   certs issued by the CA in cilium-ztunnel-secrets
       │                                          │
   verifies SPIFFE ID: spiffe://cluster.local/...
```

- Each [[ztunnel]] holds a cert **issued by the CA in `cilium-ztunnel-secrets`**.
- Node-to-node connection: trade certs, confirm "we share the same `cluster.local` trust domain."
- The **CA (`ca-root.crt`/`ca-private.key`) is the root of trust for the whole [[mTLS]] mesh** — no CA, no cert issuance, no mTLS.

### What is ztunnel — the sidecarless node proxy

[[ztunnel]] = "**zero-trust tunnel**". Originally the L4 data plane of [[Istio Ambient Mesh]].

#### Why a proxy at all — the security-guard analogy

Without a proxy, every app codes its own encryption, identity check, and access control. So you put a **"security guard"** in front of each connection. The guard handles [[mTLS]], identity verification, and access — apps stay simple. **That guard is the proxy.**

#### Old way (sidecar) — "a guard at every apartment door"

Traditional [[service mesh]] ([[Istio]] default) **injects an Envoy proxy into every pod**:

```
Sidecar model = a guard standing at every single apartment door

[Pod A][guard]   [Pod B][guard]   [Pod C][guard]
[Pod D][guard]   [Pod E][guard]   [Pod F][guard]
```

Problems:
- **100 guards** (100 pods = 100 proxies) — massive CPU/memory overhead
- Redecorate your apartment (deploy your app) → **the guard moves too** (proxy restarts with the pod)
- Tiny apartment (lightweight pod) gets the same heavy guard — out of proportion

#### New way (ztunnel) — "one guard station per building"

[[ztunnel]] runs **one proxy per node** (DaemonSet):

```
ztunnel model = one shared guard station per building

       ┌─── Node (one building) ─┐
       │ [Pod A] [Pod B] [Pod C] │
       │ [Pod D] [Pod E] [Pod F] │
       │                         │
       │   [ztunnel guard × 1]   │ ← handles all in/out for this building
       └─────────────────────────┘
```

All pod traffic on that node flows through the shared guard.

#### The three wins

| | Sidecar (guard per door) | ztunnel (guard per building) |
|---|---|---|
| **Resources** | 100 pods = 100 proxies (heavy) | 5 nodes = 5 proxies (light) |
| **Deploy impact** | Proxy restarts with app | App and proxy decoupled |
| **Ops** | Manage/upgrade 100 proxies | Manage one per node |

The second matters most. Sidecars **fuse app and security** — every app deploy shakes the security layer. [[ztunnel]] **separates security from apps** by pushing it down to the node.

#### What ztunnel actually does

- **mTLS encrypt/decrypt** (L4, **HBONE tunnel** = mTLS over HTTP/2 CONNECT)
- **Workload identity** via [[SPIFFE]]: `spiffe://cluster.local/ns/<ns>/sa/<sa>`
- **No L7** — that's [[Istio]]'s **waypoint proxy** job. [[ztunnel]] is pure L4 + mTLS.

> [!note] Trade-off
> A shared guard station ([[ztunnel]]) only sees **who's coming/going (L4)** — no fine-grained L7 inspection. We only need it for [[Cilium]]'s encryption backend, so L4 mTLS is enough.

### How Cilium + ztunnel mTLS actually flows

1. **Intercept**: [[Cilium]]'s [[eBPF]] redirects egress packets to the node's [[ztunnel]] — kernel-level, no sidecar injection.
2. **Identity + mTLS**: [[ztunnel]] uses certs issued by the `cilium-ztunnel-secrets` CA to handshake with the peer node's [[ztunnel]]. Both verify each other's [[SPIFFE]] ID (`O = cluster.local`).
3. **Tunnel**: encrypted traffic crosses the inter-node network. [[ztunnel]]'s mTLS tunnel replaces what [[IPsec]] or [[WireGuard]] would do.
4. **Decrypt + deliver**: peer [[ztunnel]] decrypts → [[eBPF]] delivers to the destination pod.

### Why the secret is bootstrap, not optional

```
cilium-ztunnel-secrets:
  ├─ ca-root.crt / ca-private.key             ← root CA that issues workload certs
  └─ bootstrap-root.crt / bootstrap-private.key  ← bootstrap cert for ztunnel itself
```

- `encryption.type: ztunnel` → [[ztunnel]] becomes a **mandatory data plane component**.
- `cilium-agent` mounts the [[ztunnel]] cert volume as `optional: false` → no secret = agent stuck in `ContainerCreating`.
- The CA is the **root for issuing workload certs**; bootstrap certs are the **initial trust anchor ztunnel uses to prove itself at startup**.
- No agent → no [[CNI]] → node `NotReady` → cluster cascade.

> [!warning] Chicken-and-egg
> For each of the 35 new clusters, run `scripts/ztunnel_generate_secret.sh` **first**. You can't bring up the data plane without the data plane's root of trust already present.

### Flannel / Calico / Cilium compared

Two questions decide it:

1. **How does inter-node pod traffic get there?** (overlay vs routing)
2. **What enforces policy/services?** (iptables vs eBPF)

#### 1. [[Flannel]] — "Ship it in a box" (VXLAN overlay)

Simplest [[CNI]]. Inter-node traffic via **overlay**.

**Problem**: node1 pod (`10.244.1.5`) → node2 pod (`10.244.2.8`). Physical network doesn't know pod IPs, only node IPs (`192.168.x.x`).

**VXLAN solution** — "box in a box":

```
Original packet:   [src pod IP → dst pod IP][data]
                           │ VXLAN wraps the whole thing
                           ▼
Encapsulated:      [node1 IP → node2 IP][ {original packet} ]
                    └─ what physical net knows ─┘
```

> [!tip] Analogy
> The mail service doesn't deliver to your office mailbox (pod IP). So put your letter in a **bigger envelope addressed to the building (node IP)**. The receiving building opens it and routes internally.

- **Pros**: works on any physical network
- **Cons**: wrap/unwrap overhead → slower
- **Policy**: none

#### 2. [[Calico]] — "Real address, direct delivery" (BGP + iptables)

**(a) Transport** — no overlay, native routing.
[[Calico]] tells routers the path to each pod IP directly, using **[[BGP]]**.

```
Flannel: wrap and ship          (overlay, slower)
Calico:  routers already know the path to pod IPs, send direct  (native, faster)
```

> [!tip] Analogy
> If [[Flannel]] is "wrap it in a bigger envelope," [[Calico]] is **"register every internal mailbox with the post office in advance"** — direct delivery, no wrapping. But the post office (physical network) must cooperate ([[BGP]] support).

**(b) Policy** — NetworkPolicy via [[iptables]]:

```
Packet arrives → iptables rule chain, top to bottom
  rule1: allow A→B?  no ↓
  rule2: drop C→D?   no ↓
  ...
  rule5000: ...      (lots of rules → linear scan → slow)
```

- **Pros**: strong/mature policy, no overlay overhead
- **Cons**: [[iptables]] degrades with rule count, depends on physical net for [[BGP]]

#### 3. [[Cilium]] — "Programs in the kernel" ([[eBPF]])

While the first two compose existing Linux tools (VXLAN, iptables), [[Cilium]] **loads small programs ([[eBPF]]) into the kernel itself** to process packets.

```
Calico(iptables):  pkt → rule1 → rule2 → ... → rule5000   (linear)
Cilium(eBPF):      pkt → eBPF hash lookup, once           (instant)
```

What [[Cilium]] adds:

| Feature | Detail |
|---|---|
| **Replaces iptables** | Even [[kube-proxy]] (service LB) becomes eBPF |
| **High perf** | Constant time regardless of rule count |
| **L7 policy** | HTTP path/method (`GET /api` allow, `DELETE` block) |
| **Identity-based** | Policy on **pod labels**, not IPs — survives IP churn |
| **Encryption** | [[IPsec]] / [[WireGuard]] / **[[ztunnel]] mTLS** built in ← what we use |

#### Side-by-side

```
                Flannel          Calico            Cilium
Inter-node      Wrapped          Direct            Direct
                (VXLAN)          (BGP routing)     (eBPF routing)
Policy engine   None             iptables          eBPF
Policy level    —                L3/L4            L3/L4/L7
Speed           moderate         fast              fastest (rule-count-independent)
Encryption      ✕                limited           IPsec/WG/ztunnel
Complexity      low              medium            high
```

**Evolution:**
- [[Flannel]] = "just make it work"
- [[Calico]] = "real policy, drop the overlay for speed"
- [[Cilium]] = "blow past [[iptables]] limits with [[eBPF]], add L7 and encryption"

> [!note] Why Cilium here
> With any other [[CNI]], we'd need a separate mesh ([[Istio]] etc.) for [[mTLS]]. [[Cilium]] **bakes encryption into the [[CNI]]**, so node-to-node traffic is auto-encrypted via [[ztunnel]] without a second control plane.

### What is the data plane

[[data plane]] = **"the actual path data packets travel."** Best understood alongside its sibling, [[control plane]].

#### The split

```
Control Plane  =  "the brain that decides"
   → builds rules: how to route, who may talk to whom

Data Plane     =  "the hands and feet that carry"
   → forwards actual packets per those rules
```

> [!tip] Post-office analogy
> - **[[control plane]]** = the dispatch room. Builds the **route table** — "letters for this region go on truck 3, those go on truck 5." Doesn't touch the mail.
> - **[[data plane]]** = the **trucks** actually carrying the mail. They follow the route table.
>
> If dispatch pauses briefly, trucks keep delivering on the existing route table. But **if the trucks stop, delivery stops.**

#### In networking

| | Control plane | Data plane |
|---|---|---|
| Role | **Decide** rules/paths | **Forward** packets |
| Frequency | Occasional (policy change) | Always (every packet) |
| Speed need | Less critical | **Very critical** — every packet flows through |
| Examples | Routing protocols, policy calc | Packet forwarding, encrypt/decrypt |

#### In Cilium

```
Control plane:  cilium-operator, policy calculation, identity management
                Computes "pod label X may talk to pod label Y" and
                writes it into eBPF maps.

Data plane:     The eBPF programs in the kernel.
                On every packet, they look up the maps and forward/drop instantly.
```

#### In this cluster — why "[[Cilium]] = the data plane itself"

```
Typical setup:
  Networking = control plane (Kubernetes) + data plane (several tools share the load)
  → one piece dying ≠ everything dies

Cilium setup:
  The data plane is entirely Cilium's eBPF
  → the path packets travel IS Cilium
```

With `encryption.type: ztunnel`, [[ztunnel]] [[mTLS]] encryption is **also part of the data plane**:

```
data plane = eBPF forwarding + ztunnel mTLS encryption
                                    │
                  cilium-ztunnel-secrets missing
                                    │
                  cilium-agent can't start
                                    │
                  No data plane gets built
                  = No path for packets
                  = Node NotReady → cluster cascade
```

### Appendix: L4 vs L7 policy

| Traffic | L7 possible? | Why |
|---|---|---|
| HTTP | ✅ | Plain text, easy parse |
| gRPC | ✅ | Rides on HTTP/2 |
| Kafka, DNS | ✅ | Cilium ships parsers |
| PostgreSQL/MySQL/Redis | ❌ → L4 | Custom binary wire format, no parser |
| TLS-encrypted traffic | ❌ → L4 | Payload is ciphertext, unreadable |
| Plain TCP/UDP streams | ❌ → L4 | No structure to parse |

Principle:
> **L7 policy is possible only when the proxy can read and understand the protocol in plaintext.**

### One-line summary

> [[Cilium]] (an [[eBPF]] [[CNI]]) redirects packets into a per-node [[ztunnel]] proxy, and [[ztunnel]] uses certs issued by the `cilium-ztunnel-secrets` CA to auto-encrypt inter-node traffic with [[mTLS]]. No sidecars means lightweight; the CA is the data plane's root of trust, so missing it kills networking outright.

## Diagram

[[canvas/Cilium과_ztunnel_eBPF_기반_CNI와_zero-trust_mTLS_터널.canvas|Concept map]]
