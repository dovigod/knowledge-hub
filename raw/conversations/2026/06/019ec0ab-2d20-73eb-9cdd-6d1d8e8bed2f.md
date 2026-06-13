---
id: 019ec0ab-2d20-73eb-9cdd-6d1d8e8bed2f
source: claude-code
created_at: '2026-06-13T11:08:24.224Z'
model: 'claude-opus-4-8[1m]'
project:
  - mvl-web-monorepo
  - infra
topics:
  - AWS infrastructure
  - Docker to cloud
  - Kubernetes EKS
  - ECR
  - HPA autoscaling
  - Deployment to Pod
  - Kubernetes networking
conversation_type:
  - learning
  - infrastructure-walkthrough
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
intent: 일단 아카이빙해
fidelity: summarized
---
<!-- kh:msg user -->
# User

Phase 2 — "노트북 끄면 죽는다" → 클라우드

- 문제: 노트북 끄면 서비스 다운, 외부 사용자 접속 불가.
- 그래서 필요: 24/7 켜진 컴퓨터 + 고정 주소
- 개념: AWS = 남의 컴퓨터 빌리기. Region/AZ, EC2(가상 서버)" 현재 infra 들 보고 뭐뭐 쓰는지 (aws) 알려줘. 이제막 로컬에서의 docker에 대한 이해를 마쳤어. 그러니 docker관련해서 중점적으로 cloud 리소스에 대해 설명해줘

<!-- kh:msg assistant -->
# Assistant

클라우드 리소스를 설명하기 전에, 실제로 이 프로젝트가 뭘 쓰는지 먼저 확인. infra 디렉토리와 Docker 관련 파일들을 탐색.

발견: 이 회사는 Kubernetes(EKS) + ArgoCD GitOps 기반. Phase 2의 "EC2 한 대" 단계를 건너뛰고 곧장 쿠버네티스(EKS)로 갔음. EC2 개념은 밑바탕 지식으로만 알면 되고, 실제로 매일 돌아가는 건 "Docker 이미지를 클라우드 위에서 대규모로 굴리는 시스템". 로컬 Docker 지식이 거의 그대로 확장됨.

## 1. 로컬 Docker에서 뭐가 바뀌나
로컬: docker build → docker run → localhost:3000 접속. 노트북 끄면 다 죽음. 클라우드는 이 3단계를 항상 켜진 남의 컴퓨터들 위로 옮긴 것.

| 로컬에서 한 일 | 클라우드에서 누가 대신 하나 | 이 회사가 쓰는 것 |
| docker build로 이미지 만들기 | CI(깃허브 액션)가 빌드 | GitHub Actions |
| 이미지를 내 디스크에 보관 | 이미지 창고(레지스트리)에 업로드 | ECR |
| docker run으로 컨테이너 실행 | 항상 켜진 서버 무리가 실행 | EKS (쿠버네티스) |
| localhost:3000 접속 | 고정 주소 + 인증서 | ALB + Route53 + Cloudflare |
| docker-compose.yml | 선언형 배포 정의 | Helm 차트 + ArgoCD |

핵심: 로컬 Docker = 내가 손으로 한다 / 클라우드 = 똑같은 일을 자동화된 시스템이 24시간 대신 한다.

## 2. Docker 이미지의 여정
① ECR — Docker 이미지 보관 창고 (Docker Hub의 AWS 버전). 실제 주소: 225989372857.dkr.ecr.ap-southeast-1.amazonaws.com/<서비스이름>:<태그>. 실제 태그 예시 arc-runner:20260529.d56d4eb (빌드 날짜 + git 커밋 해시). terraform/aws/ecr/가 정의하는 곳.
② EKS — docker run을 대신해주는 서버 무리. EC2 = 켜진 컴퓨터 한 대. EKS = EC2 여러 대를 묶어 컨테이너를 분배·실행·재시작해주는 관리자. 로컬 docker run 1개 = Pod. 죽으면 자동 재시작. HPA가 자동 개수 증가. Karpenter로 노드 자동 추가. 클러스터 여러 개: mvlchain, tada-*, onion-*, bc-* (dev/stage/prod).
③ Helm 차트 = docker-compose.yml의 확장판. common-service/values.yaml에 image.repository, tag, ports, env, livenessProbe, readinessProbe, resources 등. compose보다 추가된 게 probe(헬스체크)와 resources(자원 제한).
④ ArgoCD = Git에 적힌 대로 클러스터를 맞춰주는 자동 배포기(GitOps). Git=정답지, ArgoCD=클러스터를 정답지와 똑같이 유지하는 로봇. infra/ 디렉토리 전체가 정답지.

## 3. 외부 접속 (고정 주소)
사용자 → Cloudflare(DNS+보안) → Route53(AWS DNS) → ALB(로드밸런서) → EKS의 Pod. Route53, loadbalancer, ACM(HTTPS 인증서), Cloudflare.

## 4. AWS 전체 목록 (terraform/aws/)
컨테이너/실행: ECR, EKS, ECS, compute. 데이터: database=RDS(PostgreSQL, 컨테이너 밖에 둠), queue=SQS/MQ, storage=S3, opensearch/redshift. 네트워크: network(VPC), loadbalancer(ALB), route53, acm. 보안/인증: iam, secretmanager·ssm, cognito, sso, cloudtrail. 운영: cloudwatch, sns, events, Datadog, Cloudflare. 리전: ap-southeast-1(싱가포르 메인), us-east-1(버지니아), ap-northeast-2(서울).

다음 추천 순서: ECR → Pod=컨테이너 → Helm values.yaml↔docker-compose.yml → EKS 자동 복구.

<!-- kh:msg user -->
# User

먼저 HPA부터 설명하고, ECR 깊게 설명해줘

<!-- kh:msg assistant -->
# Assistant

실제 파일(hpa.yaml, ECR config)을 읽고 설명.

# 1부. HPA — 컨테이너 개수를 자동으로 늘렸다 줄였다
HPA(Horizontal Pod Autoscaler) = 트래픽 몰리면 Pod 개수를 자동으로 늘리고 한가하면 줄임. Horizontal(수평)=같은 컨테이너를 여러 개 띄움(일꾼 수 늘리기). Vertical(수직)=컨테이너 1개에 자원 더 줌(한계 있음).

실제 설정 (values.yaml): autoscaling.enabled: false(기본 꺼짐), minReplicas: 1(바닥), maxReplicas: 2(천장), metrics: [](판단 기준, 보통 CPU 70%), unlimitMaxReplicasCount: 150(거의 무제한 모드).

scaleTargetRef로 Deployment 또는 Rollout(카나리/블루그린)의 개수 숫자만 바꿈. 실제 Pod 생성/삭제는 Deployment가 함.

함정: HPA로 웹서버 Pod 100개로 늘려도 전부 같은 DB(RDS) 1대에 붙음. maxReplicas 천장 중요 - 무한정 늘리면 DB가 터짐. "확장은 컨테이너만 자동, DB는 별도 설계".

# 2부. ECR 깊게
ECR(Elastic Container Registry) = AWS가 운영하는 Docker 이미지 전용 비공개 창고. Docker Hub의 사설 AWS 버전. 로컬 docker build→docker push→ECR→EKS가 docker pull→Pod 실행.

EKS 서버(노드)들이 공통으로 바라보는 한 곳이 ECR. 주소 구조: 225989372857.dkr.ecr.ap-southeast-1.amazonaws.com/mvl-faucet-service:staging-a1b2c3d (계정/리전/레포이름/태그). 실제 레포 90개 넘음: mvl-clutch-server, mvl-activity-service, mvl-faucet-service, mvl-sign-service 등.

4가지 핵심 설정:
① image_tag_mutability: MUTABLE(덮어쓰기 허용, 기본)/IMMUTABLE(한번 올린 태그 못 바꿈, 프로덕션 안전).
② lifecycle_policy(가장 실용적): 오래된 이미지 자동 청소. 실제 - staging 태그 최신 15개만 유지, dist 15개, untagged는 1개 넘으면 즉시 삭제. docker image prune 자동화.
③ repository_policy: 누가 접근 가능? ArgoCD(role/management-argo-cd)와 계정 본인(225989372857:root)만 push/pull. ECR이 사설인 이유.
④ replication_rules: 싱가포르(ap-southeast-1)에 올린 TADA 이미지를 버지니아(us-east-1)로 자동 복제. registry_id 633107344074. tada-* 70여개 PREFIX_MATCH. 미국 사용자 EKS가 가까운 창고에서 빠르게, 한 리전 죽어도 살아남음.
⑤ (보너스) pull_through_cache: Docker Hub(registry-1.docker.io), k8s, quay, ghcr 등 외부 이미지를 ECR 거쳐 당겨 캐싱. cache/dockerhub/* 주소. Docker Hub rate limit 우회 + 빨라짐.

다음 추천: EKS가 ECR 이미지를 어떻게 당겨 Pod로 띄우는가 (deployment.yaml의 image: 한 줄).

<!-- kh:msg user -->
# User

엉 너 추천대로 음 단계는 "EKS가 이 ECR 이미지를 실제로 어떻게 당겨서 Pod로 띄우는가" — 즉 deployment.yaml 템플릿에서 image: 한 줄이 어떻게 Pod가 되는지예요. 보

<!-- kh:msg assistant -->
# Assistant

deployment.yaml 템플릿과 실제 배포값, ECR 인증 방식을 확인하고 설명.

# image: 한 줄 → 돌아가는 Pod까지
템플릿 핵심 두 줄: image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}", imagePullPolicy: {{ .Values.image.pullPolicy }}. Helm이 빈칸을 채워 실제로는 image: "225989...amazonaws.com/mvl-clutch-server:staging-d56d4eb", imagePullPolicy: IfNotPresent. 로컬 docker run과 똑같은 의미.

전체 7단계:
[1] Git에 image.tag 변경 커밋 → [2] ArgoCD 감지, Helm 렌더링 → [3] Deployment 객체 생성/갱신 → [4] Deployment→ReplicaSet→Pod N개 주문 → [5] 스케줄러가 어느 노드(EC2)에 놓을지 결정 → [6] 노드의 kubelet이 ECR에서 docker pull (IAM 인증) → [7] 컨테이너 실행→probe 통과→Ready→트래픽 받기.

[3~4] 3단 계층: Deployment(목표 선언)→ReplicaSet(개수 감시/유지)→Pod(실체). Pod 죽으면 ReplicaSet이 즉시 새로 생성=자기복구(self-healing), 로컬 docker run엔 없음. selector matchLabels(app, deployType: stable)로 자식 Pod 식별. HPA 켜지면 replicas를 아예 안 적어서 개수 결정권을 HPA에 넘김(둘이 안 싸우게).

[5] 스케줄러가 CPU/메모리 여유 노드 선택. nodeSelector/tolerations/affinity로 배치 제어. 남는 노드 없으면 Karpenter가 EC2 노드 자동 생성.

[6] ECR 인증 핵심: 템플릿에 imagePullSecrets(비번) 없음. 대신 노드 자체에 IAM 권한(AmazonEC2ContainerRegistryReadOnly, ecr:GetAuthorizationToken, ecr:BatchGetImage). 노드가 AWS 신분증으로 알아서 인증. repository_policy(ArgoCD/계정만 허용)와 노드 IAM이 양쪽에서 맞아떨어짐. imagePullPolicy: IfNotPresent = 이미 있으면 재다운로드 안 함.

[7] probe 3종: startupProbe(다 켜졌나, 느린 부팅용), readinessProbe(요청 받을 준비 됐나, 통과해야 트래픽), livenessProbe(살아있나, 실패시 재시작). 무중단 배포: strategy rollingUpdate maxSurge:2 maxUnavailable:0(기존 Pod 0개도 안 죽임), preStop sleep 5, terminationGracePeriodSeconds. 새 버전 Ready 확인 후 옛 버전 천천히 내림. docker stop&&run(끊김)과 결정적 차이.

다음 추천: 띄운 Pod에 외부 사용자가 어떻게 접속하나 (Service→Ingress/ALB→Route53/Cloudflare).

<!-- kh:msg user -->
# User

엉

<!-- kh:msg assistant -->
# Assistant

Service/Ingress 템플릿, TargetGroupBinding, AWS Load Balancer Controller를 확인하고 설명.

# 외부 사용자 → Pod 까지: 네트워크 전 경로
왜 Service가 필요한가: Pod는 죽고 다시 태어날 때마다 내부 IP가 바뀜. Service = 바뀌는 Pod들 앞에 세워둔 절대 안 변하는 고정 대표 주소. 로컬 localhost:3000 하나면 됐지만 클라우드는 Pod 여러 개+계속 교체라 변하지 않는 입구가 필요.

Service가 Pod 찾는 법 - label 매칭: service.yaml selector(app, deployType: stable)가 Pod labels와 일치하면 연결. 이름/IP 아니라 라벨로 느슨하게 연결돼서 Pod 죽고 새로 떠도 자동 포함. port(80)→targetPort(3000) 변환.

Service 3타입: ClusterIP(기본, 클러스터 내부만, 사내 전화), NodePort(노드 특정 포트로 외부 노출, 건물 옆문), LoadBalancer(ALB 직접 생성, 정문). 대부분 백엔드는 ClusterIP, 외부 열 것만 ALB 붙임.

외부 진입 - ALB를 Pod에 연결 2방식: AWS Load Balancer Controller(alb-ingress) 설치, serviceAccount에 eks.amazonaws.com/role-arn=IRSA로 ALB 생성/수정 권한 부여(ECR 노드 IAM과 같은 원리).
방식 A: Ingress(ingressClassName: alb) - 컨트롤러가 실제 ALB 자동 생성, ACM 인증서 붙임. 이 회사는 deprecated 표시(옛 방식).
방식 B: TargetGroupBinding(현재 선호) - ALB는 Terraform으로 미리 만들고(terraform/aws/loadbalancer), targetGroupARN에 Service Pod들 자동 등록. 인프라와 앱 관리 분리.

전체 경로: 사용자 브라우저(https) → Cloudflare(DNS+DDoS+ZeroTrust) → Route53(AWS DNS, 도메인→ALB) → ALB(HTTPS 종료 ACM 인증서+분배) → 노드(EC2) 포트 → Service(고정 입구, selector로 살아있는 Pod 선택, port80→targetPort3000) → Pod(ECR 이미지로 띄운 그 컨테이너, readinessProbe 통과한 것만).

한 바퀴 정리: 1.ECR=이미지 창고 2.EKS/Deployment/Pod=실행(자기복구) 3.HPA+Karpenter=자동 확장 4.Service→ALB→Route53/Cloudflare=외부 접속.

다음 후보: ConfigMap/Secret(환경변수·비밀값 주입, envFrom), RDS/데이터베이스(컨테이너 밖 데이터 계층), ArgoCD 배포 흐름(GitOps).

<!-- kh:msg user -->
# User

일단 아카이빙해
