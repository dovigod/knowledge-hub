---
id: 019ec0af-1f1f-760a-ada0-c5a513adba2a
title: AWS 클라우드 인프라와 Docker/EKS 배포 파이프라인
topics:
  - aws
  - ecr
  - eks
  - hpa
  - docker
  - kubernetes
  - deployment
tags:
  - aws
  - eks
  - ecr
  - kubernetes
  - docker
  - hpa
  - argocd
  - helm
  - alb
  - infra
  - learning
  - korean
sources:
  - 019ec0ab-2d20-73eb-9cdd-6d1d8e8bed2f
created_at: '2026-06-13T11:12:42.783Z'
updated_at: '2026-06-13T11:12:42.783Z'
---
> [!note] TL;DR
> 노트북 끄면 서비스가 죽으니, 24/7 켜진 [[AWS]] 컴퓨터 위로 옮긴다. 이 회사는 [[EC2]] 한 대 단계를 건너뛰고 바로 [[EKS]](쿠버네티스) 위에서 컨테이너를 굴린다. 흐름은 **[[GitHub Actions]] → [[ECR]](이미지 창고) → [[EKS]] 위의 Pod**, 외부 접속은 **사용자 → [[Cloudflare]] → [[Route53]] → [[ALB]] → [[Service]] → [[Pod]]**. [[HPA]]가 트래픽에 따라 Pod 개수를 자동 조절하고, [[Karpenter]]가 부족하면 EC2 노드를 자동 추가한다. 로컬 `docker run` 한 줄을, **자동 복구·자동 확장·고정 주소**까지 갖춘 시스템이 24시간 대신 돌려주는 셈.

## Phase 2 — "노트북 끄면 죽는다"에서 클라우드로

### 문제 정의
- 로컬 [[Docker]]: `docker build` → `docker run` → `localhost:3000`. 노트북 끄면 끝.
- 외부 사용자는 우리 집 IP를 모르고, 알아도 NAT/방화벽 뒤라 못 들어옴.
- 그래서 필요한 것은 두 가지:
  1. **24시간 켜진 컴퓨터** (실패하면 자동으로 다시 살아나는 컴퓨터들)
  2. **고정 주소** (도메인 + HTTPS)

### 개념: AWS는 "남의 컴퓨터 빌리기"
- **[[Region]]**: 지리적 위치(예: `ap-southeast-1` 싱가포르). 데이터가 물리적으로 어디 있는지.
- **[[AZ]] (Availability Zone)**: 한 Region 안의 독립 데이터센터(예: `ap-southeast-1a/b/c`). 한 AZ가 통째로 죽어도 다른 AZ는 살아남게 분산 배치.
- **[[EC2]]**: AWS가 빌려주는 가상 서버 한 대. 본질적으로 "켜진 리눅스 박스". 이 위에 뭐든 깔 수 있음.

> [!tip] 이 회사가 EC2를 직접 안 쓰는 이유
> Phase 2 교과서는 "EC2 한 대 사서 docker run" 단계가 있지만, **이 프로젝트는 그걸 건너뛰고 곧장 [[EKS]]로 갔다**. EC2 개념은 밑바탕으로만 알면 되고, 실제로 매일 도는 건 "Docker 이미지를 쿠버네티스 위에서 대규모로 굴리는 시스템"이다. 로컬 Docker 지식이 거의 그대로 확장된다.

## 로컬 Docker → 클라우드 대응표

| 로컬에서 손으로 한 일 | 클라우드에서 누가 대신 하나 | 이 회사가 쓰는 것 |
|---|---|---|
| `docker build`로 이미지 만들기 | CI가 자동 빌드 | [[GitHub Actions]] |
| 이미지를 내 디스크에 보관 | 이미지 창고(레지스트리)에 업로드 | [[ECR]] |
| `docker run`으로 컨테이너 실행 | 항상 켜진 서버 무리가 실행·재시작·분배 | [[EKS]] (쿠버네티스) |
| `localhost:3000` 접속 | 고정 도메인 + HTTPS 인증서 | [[ALB]] + [[Route53]] + [[Cloudflare]] |
| `docker-compose.yml` | 선언형 배포 정의 | [[Helm]] 차트 + [[ArgoCD]] |

> [!example] 핵심 한 줄
> **로컬 Docker** = "내가 손으로 한다" · **클라우드** = "똑같은 일을 자동화된 시스템이 24시간 대신 한다."

## Docker 이미지의 여정 (ECR → EKS → Helm → ArgoCD)

```
       (개발자)
          │ git push
          ▼
   ┌──────────────────┐
   │ GitHub Actions   │  docker build, tag, push
   └────────┬─────────┘
            │ docker push
            ▼
   ┌──────────────────────────────────────────────┐
   │ ECR  (225989372857.dkr.ecr.ap-southeast-1…)  │  ← 이미지 창고
   │   mvl-clutch-server:staging-d56d4eb          │
   │   mvl-faucet-service:20260529.d56d4eb …      │
   └────────┬─────────────────────────────────────┘
            │ image: 한 줄로 참조
            ▼
   ┌──────────────────┐         ┌────────────┐
   │ Git: infra/      │ ◀────── │ 개발자가    │
   │  Helm values.yaml│         │ tag만 바꿈 │
   │  image.tag: ...  │         └────────────┘
   └────────┬─────────┘
            │ Git이 정답지
            ▼
   ┌──────────────────┐
   │ ArgoCD           │  Git ↔ 클러스터 sync
   └────────┬─────────┘
            │ kubectl apply 자동
            ▼
   ┌──────────────────────────────────────────────┐
   │ EKS 클러스터  (EC2 노드 N대 묶음)            │
   │   Deployment → ReplicaSet → Pod              │
   │   kubelet이 ECR에서 docker pull (IAM 인증)    │
   └──────────────────────────────────────────────┘
```

### ① [[ECR]] — Docker 이미지 보관 창고
- Docker Hub의 AWS 사설 버전.
- 주소 형식: `<계정ID>.dkr.ecr.<Region>.amazonaws.com/<레포>:<태그>`
- 실제 예: `225989372857.dkr.ecr.ap-southeast-1.amazonaws.com/mvl-clutch-server:staging-d56d4eb`
- 태그 컨벤션: `20260529.d56d4eb` = `빌드날짜 + git 커밋 해시` (어떤 코드인지 추적 가능).
- 정의 위치: `terraform/aws/ecr/`.

### ② [[EKS]] — `docker run`을 대신해주는 서버 무리
- [[EC2]] = 켜진 컴퓨터 한 대.
- [[EKS]] = EC2 여러 대(노드)를 묶어 **컨테이너를 분배·실행·재시작**해주는 관리자.
- 로컬 `docker run 1개` ↔ 쿠버네티스의 [[Pod]] 1개.
- Pod 죽으면? → [[ReplicaSet]]이 즉시 새로 생성 (자기복구, self-healing).
- 트래픽 폭주? → [[HPA]]가 Pod 개수 늘림.
- 노드 부족? → [[Karpenter]]가 EC2 노드 자동 추가.
- 환경별 클러스터 분리: `mvlchain`, `tada-*`, `onion-*`, `bc-*` (dev/stage/prod).

### ③ [[Helm]] 차트 — `docker-compose.yml`의 확장판
- `common-service/values.yaml`에 `image.repository`, `tag`, `ports`, `env`, `livenessProbe`, `readinessProbe`, `resources` 등을 선언.
- compose보다 추가된 핵심 두 가지: **probe**(헬스체크)와 **resources**(자원 제한).

### ④ [[ArgoCD]] — Git이 정답지, 클러스터를 동기화
- **[[GitOps]]**: Git에 적힌 상태와 실제 클러스터를 항상 일치시킴.
- `infra/` 디렉토리 전체가 "정답지", ArgoCD는 그 정답지를 보고 클러스터를 맞추는 로봇.
- 배포 = 코드 푸시 + PR 머지. 콘솔 클릭 없음.

## 외부 접속 (고정 주소까지)

```
사용자 브라우저 (https://app.example.com)
        │
        ▼
   [[Cloudflare]]          ← DNS + DDoS 차단 + Zero Trust
        │
        ▼
   [[Route53]]              ← AWS DNS, 도메인 → ALB
        │
        ▼
   [[ALB]]                 ← HTTPS 종료 (ACM 인증서), 트래픽 분배
        │
        ▼
   EC2 노드의 포트
        │
        ▼
   [[Service]]              ← 고정 입구, label selector로 살아있는 Pod 선택
        │ port:80 → targetPort:3000
        ▼
   [[Pod]] (× N개)         ← readinessProbe 통과한 것만 트래픽 받음
```

## 이 회사가 쓰는 AWS 리소스 전체 (terraform/aws/)

| 영역 | 리소스 | 용도 |
|---|---|---|
| 컨테이너/실행 | `ecr`, `eks`, `ecs`, `compute` | 이미지 보관·컨테이너 실행 |
| 데이터 | `database`(RDS PostgreSQL), `queue`(SQS/MQ), `storage`(S3), `opensearch`, `redshift` | 영속/검색/분석 |
| 네트워크 | `network`(VPC), `loadbalancer`(ALB), `route53`, `acm` | VPC·로드밸런서·DNS·인증서 |
| 보안/인증 | `iam`, `secretmanager`, `ssm`, `cognito`, `sso`, `cloudtrail` | 권한·비밀·감사로그 |
| 운영 | `cloudwatch`, `sns`, `events`, Datadog, [[Cloudflare]] | 로그·메트릭·알람 |

> [!note] 리전 분포
> 메인은 **싱가포르 `ap-southeast-1`**, 미국 트래픽용 **`us-east-1`**(버지니아), 한국 작업용 **`ap-northeast-2`**(서울). ECR은 싱가포르 → 버지니아로 자동 복제(replication_rules)된다.

## 다이어그램

[[canvas/AWS_클라우드_인프라와_DockerEKS_배포_파이프라인.canvas|개념도]]

---

## 1부. [[HPA]] — Pod 개수를 자동��로 늘렸다 줄였다

### 정의
**HPA (Horizontal Pod Autoscaler)** = 트래픽 몰리면 Pod 개수를 자동으로 늘리고, 한가하면 줄이는 컨트롤러.

- **Horizontal (수평)**: 같은 컨테이너를 **여러 개 띄운다** = 일꾼 수 늘리기.
- **Vertical (수직)**: 컨테이너 1개에 **CPU/메모리를 더 준다** = 한 일꾼을 헬스장 보내기. 한계 있음(노드 사이즈 ceiling).

> [!tip] 왜 거의 항상 Horizontal?
> 컨테이너 자체가 stateless하게 설계돼서 "그냥 더 띄우면" 처리량이 선형으로 늘기 때문. Vertical은 큰 인스턴스 한 대에 의존하게 되고 장애 시 통째로 죽는다.

### 실제 설정 예 (values.yaml)
```yaml
autoscaling:
  enabled: false              # 기본은 꺼짐
  minReplicas: 1              # 바닥 (한가해도 이 이하로는 안 줄임)
  maxReplicas: 2              # 천장 (트래픽 와도 이 이상으론 안 늘림)
  metrics: []                 # 판단 기준 (예: CPU 평균 70%)
  unlimitMaxReplicasCount: 150   # "거의 무제한" 모드일 때의 천장
```

### 작동 메커니즘
```
Pod CPU 평균 ▶ HPA 컨트롤러 ▶ scaleTargetRef로 가서
                              Deployment.replicas 숫자만 바꿈
                                       │
                                       ▼
                              Deployment가 ReplicaSet에게
                              "Pod 개수 N개로 맞춰" 명령
                                       │
                                       ▼
                              실제 Pod 생성/삭제
```

- HPA는 **개수 숫자만 바꾼다**. 실제 Pod 생성/삭제는 Deployment(또는 Argo Rollout)가 한다.
- `scaleTargetRef`에 Deployment 이름을 적거나, 카나리/블루그린이면 [[Argo Rollouts]]를 적는다.

> [!warning] HPA의 가장 큰 함정 — DB는 같이 안 늘어난다
> 웹서버 Pod를 100개로 늘려도 **전부 같은 [[RDS]] 1대에 붙는다**. Pod만 자동 확장되고 DB는 안 늘어남.
> - `maxReplicas` 천장이 중요한 이유: 무한정 늘리면 DB 커넥션 풀이 터지고 RDS가 죽는다.
> - 원칙: **"확장은 컨테이너만 자동, DB는 별도 설계(읽기 복제본/샤딩/캐시)"**.

## 2부. [[ECR]] 깊게 — 사설 Docker 이미지 창고

### 정의와 흐름
**ECR (Elastic Container Registry)** = AWS가 운영하는 Docker 이미지 전용 **비공개** 창고. Docker Hub의 사설 AWS 버전.

```
로컬에서 docker build
        │
        ▼
docker push  ──▶  ECR (이미지 저장)
                     │
                     ▼
                 EKS 노드의 kubelet이 docker pull
                     │
                     ▼
                 Pod로 실행
```

EKS 서버(노드)들이 **공통으로 바라보는 한 곳**이 ECR. 그래서 어떤 노드에서 Pod가 떠도 같은 이미지를 쓴다.

### 주소 구조
```
225989372857 . dkr.ecr . ap-southeast-1 . amazonaws.com / mvl-faucet-service : staging-a1b2c3d
  └─ 계정ID ─┘            └─ Region ─┘                   └── 레포 이름 ──┘ └── 태그 ──┘
```

실제 레포 90개 넘음: `mvl-clutch-server`, `mvl-activity-service`, `mvl-faucet-service`, `mvl-sign-service`, …

### 4가지 핵심 설정

#### ① `image_tag_mutability`
| 값 | 의미 | 언제 |
|---|---|---|
| `MUTABLE` | 같은 태그에 덮어쓰기 가능 (기본) | dev/staging에서 빠르게 갈아끼울 때 |
| `IMMUTABLE` | 한번 올린 태그는 못 바꿈 | **프로덕션 안전** — `v1.2.3`이 어느 날 바뀌어 있을 일 없음 |

#### ② `lifecycle_policy` (가장 실용적)
오래된 이미지를 자동 청소. `docker image prune`의 자동화 버전.
```text
- staging 태그: 최신 15개만 유지, 나머지 삭제
- dist  태그: 최신 15개만 유지
- untagged: 1개 넘으면 즉시 삭제
```
없으면 ECR 비용이 무한정 늘어남.

#### ③ `repository_policy`
누가 이 레포에 접근 가능한지. 실제 정책:
- `role/management-argo-cd` (ArgoCD가 pull)
- `225989372857:root` (계정 본인 push/pull)
- **그 외는 전부 거부.** ECR이 "사설"인 이유.

#### ④ `replication_rules` (멀티 리전 복제)
싱가포르(`ap-southeast-1`)에 올린 이미지를 버지니아(`us-east-1`)로 자동 복제.
```hcl
registry_id     = "633107344074"
prefix_match    = ["tada-*"]   # 70여개 레포
destinations    = ["us-east-1"]
```
**왜?** 미국 사용자 EKS가 가까운 창고에서 빠르게 pull. 한 리전이 죽어도 다른 리전에서 살아남음.

#### ⑤ (보너스) `pull_through_cache`
Docker Hub, k8s.io, quay.io, ghcr.io의 외부 이미지를 ECR 거쳐서 당겨 캐시.
```
주소: <계정>.dkr.ecr.<region>.amazonaws.com/cache/dockerhub/library/nginx:1.25
```
- **Docker Hub rate limit 우회** (익명 풀 6시간당 100회 제한 회피).
- ECR 안에 캐시되니 다음부터는 더 빠름.

---

## 3부. `image:` 한 줄이 어떻게 Pod로 뜨는가

### 핵심 두 줄
Helm 템플릿(`deployment.yaml`):
```yaml
spec:
  containers:
  - name: app
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    imagePullPolicy: {{ .Values.image.pullPolicy }}
```

렌더링되면:
```yaml
    image: "225989372857.dkr.ecr.ap-southeast-1.amazonaws.com/mvl-clutch-server:staging-d56d4eb"
    imagePullPolicy: IfNotPresent
```

로컬 `docker run 225989...:staging-d56d4eb`과 의미적으로 동일하다. 다만 클러스터가 이걸 **자동으로, 안전하게, 여러 대에 걸쳐** 해준다.

### 전체 7단계 파이프라인

```
[1] Git에 image.tag 변경 커밋
        │
        ▼
[2] ArgoCD가 Git 변화 감지, Helm 렌더링
        │
        ▼
[3] Deployment 객체 생성/갱신
        │
        ▼
[4] Deployment → ReplicaSet → Pod N개 주문
        │
        ▼
[5] 스케줄러가 "어느 노드(EC2)에 놓을지" 결정
        │
        ▼
[6] 그 노드의 kubelet이 ECR에서 docker pull (IAM 인증)
        │
        ▼
[7] 컨테이너 실행 → probe 통과 → Ready → 트래픽 받기
```

### [3~4] 3단 계층 — Deployment → ReplicaSet → Pod

| 계층 | 역할 | 비유 |
|---|---|---|
| **Deployment** | "이런 모습의 Pod를 N개 유지해줘"라는 목표 선언 | 사장이 "직원 3명 유지" 지시 |
| **ReplicaSet** | 그 목표대로 개수를 감시·유지 | 인사팀장이 결원 발생시 즉시 채용 |
| **Pod** | 실제 컨테이너가 도는 단위 | 일하는 직원 |

```
   Deployment (replicas: 3)
        │
        ▼
   ReplicaSet (관리 중인 Pod: 3)
        │
        ├─▶ Pod A  ─ 죽음 ─▶ (ReplicaSet이 즉시 Pod D 생성)
        ├─▶ Pod B
        └─▶ Pod C
```

> [!tip] 자기복구(self-healing)
> Pod가 죽으면 ReplicaSet이 즉시 새 Pod를 만든다. 로컬 `docker run`엔 없는 능력. 이게 "노트북 끄면 죽는다" 문제를 본질적으로 해결한 것.

#### selector — 자식 Pod를 식별하는 방법
```yaml
selector:
  matchLabels:
    app: clutch-server
    deployType: stable
```
이름이 아니라 **label**로 자식 Pod를 식별. 그래서 Pod 이름이 바뀌어도 상관없음.

#### HPA가 켜졌을 때
```yaml
spec:
  # replicas: 3   ← 일부러 빼둠
```
`replicas` 필드를 아예 안 쓴다. 개수 결정권을 [[HPA]]에 통째로 넘기기 위함(둘이 서로 덮어쓰며 싸우지 않게).

### [5] 스케줄러 — 어느 EC2 노드에 놓을까

쿠버네티스 [[kube-scheduler]]가 다음을 본다:
- 노드의 CPU/메모리 여유
- Pod의 `requests` (필요 자원)
- `nodeSelector`, `tolerations`, `affinity`/`anti-affinity` (배치 제약)

남는 노드가 없으면? → **[[Karpenter]]**가 EC2 노드를 새로 띄워서 Pod를 거기 배치.

### [6] ECR에서 이미지 당기기 — IAM이 비밀번호다

> [!warning] 템플릿에 `imagePullSecrets`가 없다
> 보통 사설 레지스트리는 `docker login` 비밀번호를 Secret으로 주입한다. 이 회사 템플릿엔 그게 없다.
> 대신 **노드(EC2) 자체에 IAM 역할이 붙어 있다.**

노드 IAM 권한:
- `AmazonEC2ContainerRegistryReadOnly` (관리형 정책)
- 핵심 액션: `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`

흐름:
```
kubelet ──"ECR에서 풀해줘"──▶ AWS SDK
                                  │
                                  ▼
                             노드의 IAM 역할로 GetAuthorizationToken
                                  │
                                  ▼
                             임시 토큰으로 docker pull
```

이게 ECR의 `repository_policy`(ArgoCD/계정만 허용)와 **양쪽에서 맞아떨어진다**:
- ECR 쪽: "이 IAM principal만 받음"
- 노드 쪽: "내 IAM이 그 principal이다"

#### `imagePullPolicy`
| 값 | 의미 |
|---|---|
| `Always` | 매번 ECR에서 다시 풀 |
| `IfNotPresent` | 노드에 이미 있으면 재다운로드 안 함 (이 회사 기본) |
| `Never` | 절대 풀 안 함 (이미 있는 것만 씀) |

### [7] Probe와 무중단 배포

#### 3종 probe

| 종류 | 묻는 것 | 실패하면 |
|---|---|---|
| `startupProbe` | "다 켜졌어?" | 다른 probe 시작을 미룸. 부팅 느린 앱용 |
| `readinessProbe` | "요청 받을 준비 됐어?" | Service에서 빼서 트래픽 안 보냄 |
| `livenessProbe` | "살아있어?" | Pod **재시작** |

예시:
```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
```

#### 무중단 배포(Rolling Update)
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 2          # 새 Pod를 최대 2개까지 미리 띄움
    maxUnavailable: 0    # 기존 Pod는 0개도 안 죽임 (= 항상 풀 가용)
```

플러스:
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]   # 종료 직전 5초 버퍼 (in-flight 요청 처리)
terminationGracePeriodSeconds: 30
```

흐름:
```
old Pod ×3   ── 새 Pod 시작 ── new Pod 시작 (Ready 대기)
                                   │
                                   ▼ Ready!
old Pod ×3 ───── 트래픽 일부 new Pod로
                                   │
                                   ▼
old Pod ×2 + new Pod ×1 (점진적으로 교체)
                                   │
                                   ▼
new Pod ×3 (배포 완료, 다운타임 0)
```

> [!example] 로컬 `docker stop && docker run`과의 결정적 차이
> 로컬에선 컨테이너를 멈추는 순간 요청이 끊긴다. 쿠버네티스는 **새 버전이 Ready인 걸 확인한 뒤에** 옛 버전을 천천히 내린다. 사용자는 배포가 일어났는지도 모른다.

## 한 바퀴 정리

```
1. ECR              = 이미지 창고
2. EKS/Deployment   = 실행 + 자기복구
3. HPA + Karpenter  = 자동 확장 (Pod ↑, 노드 ↑)
4. Service → ALB    = 외부 접속
5. Route53 + CF     = 도메인 + 보안
6. ArgoCD           = Git 정답지로 자동 동기화
```

> [!tip] 다음 후보 주제
> - **[[ConfigMap]] / [[Secret]]**: 환경변수·비밀값 주입 (`envFrom`)
> - **[[RDS]] / 데이터베이스 계층**: 왜 컨테이너 밖에 두는가
> - **[[ArgoCD]] GitOps 배포 흐름**: PR → sync → rollout
> - **[[IRSA]]**: Pod 수준의 AWS 권한 (노드 IAM과 다름)

---

## English

> [!note] TL;DR
> When your laptop shuts off, the service dies. The fix: move everything onto AWS computers that run 24/7. This company skips the "single [[EC2]] box" step and runs containers directly on [[EKS]] (Kubernetes). The pipeline is **[[GitHub Actions]] → [[ECR]] (image registry) → Pods on [[EKS]]**, and external traffic flows **user → [[Cloudflare]] → [[Route53]] → [[ALB]] → [[Service]] → [[Pod]]**. [[HPA]] auto-scales Pod count based on traffic; [[Karpenter]] adds EC2 nodes when capacity runs out. In essence, a single local `docker run` is replaced by a system that runs that container 24/7 with **self-healing, auto-scaling, and a stable address**.

### Phase 2 — From "laptop off = down" to the cloud

#### The problem
- Local [[Docker]]: `docker build` → `docker run` → `localhost:3000`. Laptop off = dead.
- External users don't know your home IP, and even if they did, NAT/firewall blocks them.
- Two things needed:
  1. A computer that's **on 24/7** (and respawns itself when broken).
  2. A **stable address** (domain + HTTPS).

#### Concept: AWS = renting someone else's computer
- **[[Region]]**: a geographic location (e.g., `ap-southeast-1` Singapore). Where your data physically lives.
- **[[AZ]] (Availability Zone)**: an independent data center inside a region (`ap-southeast-1a/b/c`). Spread across AZs so one going down doesn't kill you.
- **[[EC2]]**: a virtual server you rent — basically a Linux box that's on. Install anything.

> [!tip] Why this company doesn't use raw EC2
> Textbook Phase 2 has a "rent one EC2, `docker run`" step. **This project skips it and goes straight to [[EKS]]**. EC2 stays as background knowledge; the day-to-day reality is "a system that runs Docker images at scale on top of Kubernetes." Your local Docker knowledge transfers almost verbatim.

### Local Docker → Cloud, mapped

| What you did locally by hand | Who does it in the cloud | This company's pick |
|---|---|---|
| `docker build` the image | CI builds automatically | [[GitHub Actions]] |
| Store image on local disk | Upload to a registry | [[ECR]] |
| `docker run` the container | A pool of always-on servers runs/restarts/distributes | [[EKS]] (Kubernetes) |
| Hit `localhost:3000` | Stable domain + HTTPS cert | [[ALB]] + [[Route53]] + [[Cloudflare]] |
| `docker-compose.yml` | Declarative deployment spec | [[Helm]] chart + [[ArgoCD]] |

> [!example] The one-line takeaway
> **Local Docker** = "I do it by hand." · **Cloud** = "Same job, done 24/7 by an automated system."

### The journey of a Docker image (ECR → EKS → Helm → ArgoCD)

```
       (developer)
          │ git push
          ▼
   ┌──────────────────┐
   │ GitHub Actions   │  docker build, tag, push
   └────────┬─────────┘
            │ docker push
            ▼
   ┌──────────────────────────────────────────────┐
   │ ECR  (225989372857.dkr.ecr.ap-southeast-1…)  │  ← image store
   │   mvl-clutch-server:staging-d56d4eb          │
   │   mvl-faucet-service:20260529.d56d4eb …      │
   └────────┬─────────────────────────────────────┘
            │ referenced by `image:` line
            ▼
   ┌──────────────────┐         ┌────────────┐
   │ Git: infra/      │ ◀────── │ dev bumps  │
   │  Helm values.yaml│         │ the tag    │
   │  image.tag: ...  │         └────────────┘
   └────────┬─────────┘
            │ Git is the source of truth
            ▼
   ┌──────────────────┐
   │ ArgoCD           │  Git ↔ cluster sync
   └────────┬─────────┘
            │ kubectl apply (automated)
            ▼
   ┌──────────────────────────────────────────────┐
   │ EKS cluster (N EC2 nodes)                    │
   │   Deployment → ReplicaSet → Pod              │
   │   kubelet pulls from ECR (IAM auth)          │
   └──────────────────────────────────────────────┘
```

#### ① [[ECR]] — image warehouse
- AWS's private Docker Hub.
- Address: `<accountID>.dkr.ecr.<region>.amazonaws.com/<repo>:<tag>`
- Real example: `225989372857.dkr.ecr.ap-southeast-1.amazonaws.com/mvl-clutch-server:staging-d56d4eb`
- Tag convention: `20260529.d56d4eb` = `build date + git commit SHA` (traceable to code).
- Defined in `terraform/aws/ecr/`.

#### ② [[EKS]] — the pool that replaces your `docker run`
- [[EC2]] = one box that's on.
- [[EKS]] = a manager that **schedules, runs, and restarts containers** across many EC2 nodes.
- Local `docker run` ↔ one [[Pod]] in Kubernetes.
- Pod dies? → [[ReplicaSet]] spawns a new one (self-healing).
- Traffic spike? → [[HPA]] adds Pods.
- Out of node capacity? → [[Karpenter]] adds EC2 nodes.
- Clusters per env: `mvlchain`, `tada-*`, `onion-*`, `bc-*` (dev/stage/prod).

#### ③ [[Helm]] chart — `docker-compose.yml` on steroids
- `common-service/values.yaml` declares `image.repository`, `tag`, `ports`, `env`, `livenessProbe`, `readinessProbe`, `resources`, …
- Two key additions over compose: **probes** (health checks) and **resources** (limits).

#### ④ [[ArgoCD]] — Git is the source of truth
- **[[GitOps]]**: the cluster is continuously reconciled to match Git.
- The whole `infra/` directory is the "answer key"; ArgoCD is the robot that makes the cluster look like it.
- Deploy = commit + merge. No console clicking.

### External access (to a stable address)

```
User browser (https://app.example.com)
        │
        ▼
   [[Cloudflare]]         ← DNS + DDoS + Zero Trust
        │
        ▼
   [[Route53]]            ← AWS DNS, domain → ALB
        │
        ▼
   [[ALB]]                ← HTTPS termination (ACM cert), routing
        │
        ▼
   EC2 node port
        │
        ▼
   [[Service]]            ← stable entry, selects live Pods by label
        │ port:80 → targetPort:3000
        ▼
   [[Pod]] (× N)          ← only readinessProbe-passing Pods get traffic
```

### Full AWS resource inventory (terraform/aws/)

| Area | Resource | Purpose |
|---|---|---|
| Containers / compute | `ecr`, `eks`, `ecs`, `compute` | Image storage + container execution |
| Data | `database` (RDS PostgreSQL), `queue` (SQS/MQ), `storage` (S3), `opensearch`, `redshift` | Persistence / search / analytics |
| Network | `network` (VPC), `loadbalancer` (ALB), `route53`, `acm` | VPC / LB / DNS / TLS cert |
| Security / identity | `iam`, `secretmanager`, `ssm`, `cognito`, `sso`, `cloudtrail` | Authz / secrets / audit |
| Operations | `cloudwatch`, `sns`, `events`, Datadog, [[Cloudflare]] | Logs / metrics / alerting |

> [!note] Region layout
> Primary in **Singapore (`ap-southeast-1`)**, US traffic in **`us-east-1`** (Virginia), Korean engineering in **`ap-northeast-2`** (Seoul). ECR auto-replicates Singapore → Virginia via `replication_rules`.

---

### Part 1. [[HPA]] — auto-scaling Pod count

#### Definition
**HPA (Horizontal Pod Autoscaler)** automatically grows the Pod count when traffic rises and shrinks it when traffic falls.

- **Horizontal**: spin up **more copies** of the same container — more workers.
- **Vertical**: give **more CPU/memory** to one container — send one worker to the gym. Capped by node size.

> [!tip] Why horizontal almost always wins
> Containers are designed stateless, so "just run more of them" scales throughput linearly. Vertical scaling concentrates risk into one large box that dies all at once.

#### Real config example (values.yaml)
```yaml
autoscaling:
  enabled: false              # off by default
  minReplicas: 1              # floor (won't go below even when idle)
  maxReplicas: 2              # ceiling (won't go above even under load)
  metrics: []                 # signals (typically avg CPU 70%)
  unlimitMaxReplicasCount: 150   # ceiling for the "essentially unlimited" preset
```

#### How it actually works
```
Pod CPU avg ▶ HPA controller ▶ updates Deployment.replicas via
                               scaleTargetRef
                                       │
                                       ▼
                               Deployment tells the ReplicaSet
                               "I want N Pods"
                                       │
                                       ▼
                               ReplicaSet creates/deletes Pods
```

- HPA **only changes the number**. Actual Pod create/delete is done by the Deployment (or [[Argo Rollouts]] for canary/blue-green).
- `scaleTargetRef` points to either a Deployment or a Rollout.

> [!warning] HPA's biggest gotcha — the DB doesn't scale with you
> Scale your web Pods to 100 and **all 100 still hit one [[RDS]] instance**. Pods scale; the database doesn't.
> - That's why `maxReplicas` matters: unbounded scaling will exhaust the DB connection pool and kill RDS.
> - Principle: **"Only the container layer auto-scales. The DB needs separate design (read replicas, sharding, caching)."**

### Part 2. [[ECR]] deep dive — private Docker image registry

#### Definition and flow
**ECR (Elastic Container Registry)** is AWS's **private** Docker image registry — a managed, account-scoped Docker Hub.

```
local docker build
        │
        ▼
docker push  ──▶  ECR (stores image)
                     │
                     ▼
                 kubelet on an EKS node does docker pull
                     │
                     ▼
                 Pod runs
```

ECR is the **single source** that every EKS node looks at — so wherever a Pod lands, it pulls the same image.

#### Address anatomy
```
225989372857 . dkr.ecr . ap-southeast-1 . amazonaws.com / mvl-faucet-service : staging-a1b2c3d
  └─ acct ID ─┘            └── region ──┘                  └─── repo ────┘ └─── tag ────┘
```

90+ real repos: `mvl-clutch-server`, `mvl-activity-service`, `mvl-faucet-service`, `mvl-sign-service`, …

#### 4 core settings

##### ① `image_tag_mutability`
| Value | Meaning | When |
|---|---|---|
| `MUTABLE` | Overwriting a tag is allowed (default) | dev/staging, fast iteration |
| `IMMUTABLE` | Once pushed, a tag is locked | **production safety** — `v1.2.3` will never silently change |

##### ② `lifecycle_policy` (the most practical one)
Auto-prune old images — `docker image prune` automated.
```text
- tag staging: keep newest 15, delete the rest
- tag dist:    keep newest 15
- untagged:    delete if more than 1 exists
```
Without it, ECR storage cost grows unbounded.

##### ③ `repository_policy`
Who can touch this repo. Actual policy here:
- `role/management-argo-cd` (ArgoCD pulls)
- `225989372857:root` (the account itself pushes/pulls)
- **Everyone else denied.** That's why "private."

##### ④ `replication_rules` (cross-region replication)
Singapore (`ap-southeast-1`) → Virginia (`us-east-1`) auto-replication.
```hcl
registry_id     = "633107344074"
prefix_match    = ["tada-*"]   # ~70 repos
destinations    = ["us-east-1"]
```
**Why?** US-side EKS pulls from a nearby registry → faster startup; survives a regional outage.

##### ⑤ (bonus) `pull_through_cache`
Pull images from Docker Hub / k8s.io / quay.io / ghcr.io **through ECR**, cached.
```
URL: <acct>.dkr.ecr.<region>.amazonaws.com/cache/dockerhub/library/nginx:1.25
```
- Sidesteps **Docker Hub rate limits** (anonymous: 100 pulls / 6h).
- Cached inside ECR → faster on subsequent pulls.

---

### Part 3. How one `image:` line becomes a running Pod

#### The two key lines
Helm template (`deployment.yaml`):
```yaml
spec:
  containers:
  - name: app
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    imagePullPolicy: {{ .Values.image.pullPolicy }}
```

After rendering:
```yaml
    image: "225989372857.dkr.ecr.ap-southeast-1.amazonaws.com/mvl-clutch-server:staging-d56d4eb"
    imagePullPolicy: IfNotPresent
```

Semantically identical to a local `docker run 225989...:staging-d56d4eb` — except the cluster does it **automatically, safely, and across many nodes**.

#### The full 7-step pipeline

```
[1] Commit image.tag change in Git
        │
        ▼
[2] ArgoCD notices, renders Helm
        │
        ▼
[3] Deployment object created/updated
        │
        ▼
[4] Deployment → ReplicaSet → orders N Pods
        │
        ▼
[5] Scheduler picks which EC2 node each Pod lands on
        │
        ▼
[6] That node's kubelet does docker pull from ECR (IAM auth)
        │
        ▼
[7] Container starts → probes pass → Ready → receives traffic
```

#### [3-4] Three-tier hierarchy — Deployment → ReplicaSet → Pod

| Tier | Role | Analogy |
|---|---|---|
| **Deployment** | Declares the goal: "keep N Pods that look like this" | CEO says "we always have 3 staff" |
| **ReplicaSet** | Watches and enforces the count | HR lead hires immediately when someone quits |
| **Pod** | The actual running container | The employee doing work |

```
   Deployment (replicas: 3)
        │
        ▼
   ReplicaSet (managing 3 Pods)
        │
        ├─▶ Pod A  ─ dies ─▶ (ReplicaSet immediately spawns Pod D)
        ├─▶ Pod B
        └─▶ Pod C
```

> [!tip] Self-healing
> When a Pod dies, the ReplicaSet creates a fresh one instantly. Local `docker run` has no equivalent. This is the structural fix for "laptop-off-means-dead."

##### selector — how parents find their children
```yaml
selector:
  matchLabels:
    app: clutch-server
    deployType: stable
```
Children are identified by **labels**, not by name. Pod names can churn freely.

##### When HPA is on
```yaml
spec:
  # replicas: 3   ← intentionally omitted
```
The `replicas` field is left out so [[HPA]] owns the count (otherwise the two controllers would fight each other writing replicas back and forth).

#### [5] The scheduler — choosing an EC2 node

Kubernetes' [[kube-scheduler]] considers:
- Node CPU/memory headroom
- Pod `requests` (resource demands)
- `nodeSelector`, `tolerations`, `affinity`/`anti-affinity` (placement constraints)

No free node? → **[[Karpenter]]** provisions a fresh EC2 node on the fly and the Pod lands there.

#### [6] Pulling from ECR — IAM is the password

> [!warning] No `imagePullSecrets` in the template
> Normally a private registry requires a `docker login` secret. This company's template has none.
> Instead, **the IAM role attached to the EC2 node provides the credentials.**

Node IAM permissions:
- `AmazonEC2ContainerRegistryReadOnly` (managed policy)
- Key actions: `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer`

Flow:
```
kubelet ──"please pull from ECR"──▶ AWS SDK
                                       │
                                       ▼
                                  Node IAM role → GetAuthorizationToken
                                       │
                                       ▼
                                  Short-lived token → docker pull
```

This dovetails with the ECR `repository_policy` (only ArgoCD + the account allowed):
- ECR side: "only this IAM principal is allowed"
- Node side: "my IAM matches that principal"

##### `imagePullPolicy`
| Value | Meaning |
|---|---|
| `Always` | Always pull from ECR |
| `IfNotPresent` | Skip pull if already cached on node (this company's default) |
| `Never` | Never pull — must already exist locally |

#### [7] Probes and zero-downtime deploys

##### Three probe types

| Type | Question | On failure |
|---|---|---|
| `startupProbe` | "Are you up yet?" | Delays other probes. For slow-boot apps |
| `readinessProbe` | "Ready for traffic?" | Pulled out of Service endpoints |
| `livenessProbe` | "Still alive?" | Pod is **restarted** |

Example:
```yaml
readinessProbe:
  httpGet:
    path: /healthz
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
```

##### Zero-downtime rolling updates
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 2          # bring up to 2 extra new Pods before tearing down
    maxUnavailable: 0    # never kill an old Pod before its replacement is Ready
```

Plus:
```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]   # buffer to drain in-flight requests
terminationGracePeriodSeconds: 30
```

Flow:
```
old Pod ×3   ── start new Pod ── new Pod starting (waiting for Ready)
                                       │
                                       ▼ Ready!
old Pod ×3 ───── some traffic routed to new Pod
                                       │
                                       ▼
old Pod ×2 + new Pod ×1 (gradual cutover)
                                       │
                                       ▼
new Pod ×3 (deploy done, zero downtime)
```

> [!example] The decisive difference vs. local `docker stop && docker run`
> Locally, the moment you stop a container, requests drop. Kubernetes verifies the **new version is Ready** before draining the old. Users don't notice a deploy happened.

### One-loop summary

```
1. ECR              = image store
2. EKS/Deployment   = execution + self-healing
3. HPA + Karpenter  = auto-scale (Pods ↑, nodes ↑)
4. Service → ALB    = external entry
5. Route53 + CF     = domain + edge security
6. ArgoCD           = Git-driven reconciliation
```

> [!tip] Next candidate topics
> - **[[ConfigMap]] / [[Secret]]**: injecting env vars and secrets (`envFrom`)
> - **[[RDS]] / data tier**: why the DB lives outside containers
> - **[[ArgoCD]] GitOps flow**: PR → sync → rollout
> - **[[IRSA]]**: per-Pod AWS permissions (distinct from node IAM)

## Diagram

[[canvas/AWS_클라우드_인프라와_DockerEKS_배포_파이프라인.canvas|Concept map]]
