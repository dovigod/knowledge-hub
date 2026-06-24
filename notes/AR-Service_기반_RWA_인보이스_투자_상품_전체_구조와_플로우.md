---
id: 019ef773-fa95-775c-9848-9705d91bf551
title: AR-Service 기반 RWA 인보이스 투자 상품 전체 구조와 플로우
topics:
  - ar-service
  - rwa
  - invoice
  - apr
  - 보증
  - 담보
  - escrow
  - tranche
  - pricing
  - default
  - payment
  - actors
  - 수익모델
tags:
  - ar-service
  - RWA
  - accounts-receivable
  - blockchain
  - nestjs
  - 교육용
  - 한국어
sources:
  - 019ef25a-24bd-71ab-9f50-91d36bb5ea6b
created_at: '2026-06-24T02:27:13.685Z'
updated_at: '2026-06-24T02:27:13.685Z'
---
## TL;DR

[[AR-Service]]는 [[RWA]] **인보이스(매출채권)** 를 토큰화·조각화해서 투자 상품으로 만드는 플랫폼이다. 등장 액터는 **5명** — [[운영자]](Operator)가 화이트리스트 등록을 수행하고, [[판매자]](Seller)는 외상 채권을 팔아 즉시 현금화하고, [[지급인]](Payer)은 만기에 다달이 갚고, [[투자자]](Investor)는 조각을 사서 차익([[APR]])을 노리고, [[보증인]](Guarantor)은 디폴트 위험을 떠안는 대가로 보증료를 챙긴다. 실제 자금은 **온체인 금고 3형제** — [[Guarantor Vault]](담보), [[Purchase Escrow]](투자자→판매자 구매대금), [[Payback Escrow]](지급인→투자자 상환금) — 가 보관하며, [[AR-Service]]는 돈을 직접 만지지 않고 **기록·조율자**로서 IPFS 업로드·온체인 등록·아웃박스 재시도·정산 이벤트 디코딩만 수행한다. **보증 유무**가 위험·수익 구조의 축이라 [[isGuaranteed=false]]면 투자자가 손실을 전부 떠안는 대신 [[APR]]이 높고, [[isGuaranteed=true]]면 [[coverageRatio]]만큼 [[Guarantor Vault]] 담보로 메워주는 대신 [[APR]]은 낮고 이익에서 [[guaranteeFee]]가 떼인다.

> [!note] 한 줄 요약
> **[[AR-Service]] = 매출채권을 NFT 조각으로 쪼개 파는 시장의 "공책 비서"이고, 진짜 돈은 [[온체인 금고 3형제]]가 들고 있으며, [[보증인]]이 끼면 안전·저수익, 안 끼면 위험·고수익.**

> [!example] 30초 그림
> ```
>                      ┌──────────────────────────────┐
>                      │   AR-Service (기록·조율자)    │
>                      │  IPFS · Outbox · Aggregator  │
>                      └───────┬───────────────┬──────┘
>                              │ register      │ sync
>                              ▼               ▼
>   [Seller] ──외상채권──▶ [Invoice 10조각] ──▶ [Investor]
>      ▲                         │                 │
>      │ 즉시 현금화              │ 디폴트          │ 만기 redeem
>      │                         ▼                 ▼
>   [Purchase Escrow] ◀── [Payback Escrow] ◀── [Payer] 다달이 상환
>                                    ▲
>                                    │ 부족분 보전 (coverageRatio %)
>                                    │
>                            [Guarantor Vault] ◀── [Guarantor] (담보 예치)
> ```

| 축 | [[비보증]] (`isGuaranteed=false`) | [[보증]] (`isGuaranteed=true`) |
|---|---|---|
| 위험 부담 | 투자자 전액 | [[Guarantor]]가 [[coverageRatio]]까지 보전 |
| [[APR]] | 높음 | 낮음 |
| 리스팅 경로 | SC 등록 즉시 `LISTED` | `GuaranteePosition` 온체인 등록 성공 후 `LISTED` |
| 추가 수수료 | 없음 | [[guaranteeFee]] = 투자자 이익 × fee% |
| 디폴트 시 | 손실 그대로 | 컨트랙트가 [[Guarantor Vault]] 담보 인출 |

## 핵심 용어 정리 (APR · Invoice · Fraction · IPFS 번호표)

이 노트 전체에서 반복적으로 등장하는 네 가지 단어를 먼저 못박아 두자. 이 단어들의 정의가 흔들리면 뒤에 나오는 [[가격-전략]], [[보증-구조]], [[redeem-창구]] 이야기가 전부 미궁이 된다.

## 다이어그램

[[canvas/AR-Service_기반_RWA_인보이스_투자_상품_전체_구조와_플로우.canvas|개념도]]

---

### 1) APR (Annual Percentage Rate, 단리 연이율)

> [!note] 한 줄 정의
> **APR = 1년 기준 단리 이자율.** 복리(이자에 이자가 또 붙는 효과)는 빼고 계산한, "순수한 1년치 수익률(또는 대출 비용률)".

가장 단순한 예:

```
100만원을 1년 예치 → 1년 뒤 5만원 받음
APR = 5만원 / 100만원 = 5%
```

[[ar-service]]의 [[Invoice]]에는 `apr` 값이 직접 저장되거나, [[가격-전략]] 안의 `minApr`/`maxApr`로 표현된다. 사람용 표기는 퍼센트(예: `7.5`), 온체인으로 보낼 때는 ×100 해서 **bps(basis points, 750)** 로 변환한다 — 부동소수 오차를 차단하기 위함이다 (`invoice-onchain.service.ts:127`).

#### APR vs APY — 헷갈리지 말 것

| 항목 | APR | APY |
|---|---|---|
| 이자 계산 방식 | **단리** (simple) | **복리** (compound) |
| 복리 효과 | 무시 | 반영 |
| 같은 명목값일 때 | 더 낮게 표시됨 | 더 높게 표시됨 |
| AR-Service에서 쓰는 것 | ✅ APR | ❌ |

> [!tip] AR 인보이스에서 APR이 단리여도 충분한 이유
> 인보이스 한 장의 만기는 보통 **30~90일**, 길어도 1년 안쪽이다. 짧은 만기에서는 복리/단리 차이가 미미하므로 단리 APR로 충분하고, 무엇보다 **만기 일시 상환** 구조라서 이자가 중간에 재투자되지 않는다. 즉 복리 효과 자체가 생기지 않는다.

#### 만기까지 거리에 따라 APR이 어떻게 살아 움직이는가

같은 액면가 [[Invoice]]라도 **지금 사느냐 / 만기 직전에 사느냐**에 따라 적용되는 APR이 달라진다. 만기에 가까울수록 위험을 덜 떠안으므로 APR이 낮아지는 게 자연스럽다 — 이것이 뒤에 나오는 [[LINEAR_DECAY]], [[STEP_DECAY]] 전략의 핵심이다.

---

### 2) Invoice — 토큰화된 매출채권 (단순 청구서가 아님)

> [!warning] "Invoice = 청구서" 라는 통념은 절반만 맞다
> [[ar-service]] 문맥에서 **Invoice는 "토큰화되어 거래되는 매출채권(AR, Accounts Receivable)"** 이다. 종이 한 장이 아니라, **온체인에 등록되어 조각으로 쪼개져 팔리는 금융 상품**이다.

#### 본질 — "나중에 받을 권리"를 미리 파는 것

```
[Seller] ── 물건/서비스 판매 ──> [Payer]
   │                                  │
   └── 60일 뒤 1,000원 받을 권리 ◀────┘
         (이게 매출채권 = Invoice)
                 │
                 ▼
   "지금 950원 받고 만기 권리를 너한테 넘길게"
                 │
                 ▼
          [Investor] 가 950원에 매입
                 │
        60일 뒤 1,000원 회수 → +50원 = APR
```

판매자([[Seller]])는 만기까지 기다리지 않고 **즉시 현금**을 얻고, 투자자([[Investor]])는 만기 액면가와 매입가의 **할인 차익**을 얻는다. 이 사이를 [[ar-service]]가 기록·조율한다.

#### 주요 필드 (DB schema 기준)

| 필드 | 타입 | 의미 |
|---|---|---|
| `invoiceId` | Int (100000부터) | 시스템 식별자 |
| `faceValue` | Decimal(20,6) | **만기에 갚을 액면가** |
| `dueDate` | DateTime | 만기일 |
| `fractionCount` / `availableCount` | Int | 전체 조각 수 / 남은 조각 |
| `isGuaranteed` | Boolean | [[보증-구조]] 적용 여부 |
| `strategy` / `strategyParams` | Enum / JSON | [[가격-전략]] (FIXED_APR / LINEAR_DECAY / STEP_DECAY) |
| `ipfsCid` | String | [[IPFS]] 메타데이터 번호표 — 온체인 등록 전 필수 |
| `txHash` | String | 온체인 등록 트랜잭션 해시 |
| `tokenIds` | Int[] | 조각마다 발급된 NFT 토큰 ID 목록 |

#### 상태 머신 — 두 갈래의 신호등

Invoice는 **"판매 진행 상태"** 와 **"상환 진행 상태"** 두 축으로 동시에 움직인다.

```
[판매축] InvoiceSavedStatus
  DRAFT ──► CONFIRMED ──► LISTED ──► PARTIALLY_FUNDED ──► SOLD_OUT
  (생성)   (확정/IPFS)  (진열대)    (일부 판매)         (전부 판매)

[상환축] InvoicePaymentStatus
  UNPAID ──► PARTIALLY_PAID ──► PAID         (해피엔딩)
       └──► OVERDUE ──► MISSED ──► DEFAULTED  (디폴트)
```

> [!example] 왜 두 축인가
> "투자자가 다 샀느냐(판매축)"와 "지급인이 갚았느냐(상환축)"는 완전히 독립적인 사건이다. `SOLD_OUT && PARTIALLY_PAID`도, `LISTED && UNPAID`도 자연스러운 조합이다.

---

### 3) Fraction — 왜 케이크처럼 조각으로 나누는가

[[Invoice]] 한 장은 통째로 한 명에게 팔지 않고 **`fractionCount`** 개의 조각으로 잘려서 팔린다. 예를 들어 액면가 1,000원짜리 쪽지를 10조각으로 자르면 한 조각 = 100원이다.

```
            Invoice (faceValue 1,000원)
                       │
       ┌───────┬───────┼───────┬───────┐
       ▼       ▼       ▼       ▼       ▼
     조각1   조각2   조각3   ...   조각10
     (100원) (100원) (100원)       (100원)
       │       │       │             │
       ▼       ▼       ▼             ▼
   투자자A 투자자A  투자자B        투자자C
   (한 명이 여러 조각 보유 가능 — 1:N 관계)
```

#### 왜 굳이 쪼개나? 세 가지 이유

> [!tip] 조각화의 세 가지 이득
> 1. **소액 참여** — 1,000원짜리 채권에 100원만으로도 투자 가능 → 진입장벽 ↓
> 2. **위험 분산** — 투자자는 한 쪽지에 올인하지 않고 여러 쪽지·여러 조각으로 분산
> 3. **디지털 도장(NFT)** — 조각 하나하나가 온체인 토큰(`tokenIds Int[]`). 누가 어느 조각을 가졌는지가 마법 공책(블록체인)에 박혀 있어 위변조·이중판매 불가

#### 조각 가격 공식

```
조각 1개 가격 = faceValue ÷ fractionCount
            (구매 시점의 APR로 할인된 가격)

예) faceValue 1,000원, fractionCount 10
    조각당 액면가 = 100원
    APR 적용 후 매입가 = 95원 (만기 30일, APR ~60% 가정)
```

판매 진행 상태는 `availableCount`로 추적된다. 모든 조각이 팔리면 `SOLD_OUT`, 일부만 팔리면 `PARTIALLY_FUNDED`. `LISTED → PARTIALLY_FUNDED → SOLD_OUT`의 자동 전이는 조각이 한 개씩 팔릴 때마다 일어난다.

---

### 4) ipfsCid — 인터넷 사물함 번호표

> [!note] 한 줄 정의
> **ipfsCid는 IPFS(InterPlanetary File System)에 올린 Invoice 메타데이터의 보관증 ID.** 마법 공책([[블록체인]])에는 무거운 메타데이터를 직접 박지 않고 **이 번호표만** 적어둔다.

#### 왜 IPFS인가, 왜 번호표만 박나

```
[Invoice 메타데이터: faceValue, dueDate, paymentMethod, ...]
                       │
                       ▼
            [Pinata IPFS] 에 업로드
                       │
                       ▼
        ipfsCid 받음: "QmXk7Ya..."  ← 번호표
                       │
                       ▼
       Blockchain에는 이 번호표만 박음
       (메타데이터 본문은 IPFS에 머묾)
```

> [!tip] 왜 본문을 온체인에 안 박는가
> 1. **가스비** — 블록체인에 큰 JSON을 저장하면 비싸다. 번호표(32바이트) 하나만 박으면 싸다.
> 2. **불변 증명** — IPFS는 CID 자체가 내용의 해시. CID가 같다 = 내용이 1비트도 안 변했다는 수학적 보장.
> 3. **위변조 차단** — 누가 IPFS의 파일을 바꾸면 CID가 바뀌므로 온체인 CID와 불일치 → 즉시 들통.

#### "번호표 없으면 온체인 등록 불가" 라는 게이트

[[ar-service]]의 처리 파이프라인에서 `ipfsCid`는 **명시적 전제조건**이다:

```
generateInvoices
   → confirmAllInvoices (DRAFT → CONFIRMED)
   → IPFS 업로드 → ipfsCid 획득   ◀── 이 단계 없으면 다음으로 못 감
   → SC Register (온체인 등록)
   → (보증이면 GuaranteePosition 등록)
   → LISTED (진열대 등장)
```

> [!warning] ipfsCid가 필수인 진짜 이유
> 온체인 등록(`SC_REGISTER`) 트랜잭션의 입력값 자체에 `ipfsCid`가 들어간다. 번호표 없이는 컨트랙트 호출 자체가 불가능하다. 즉 **"진짜 채권 메타데이터가 어딘가에 영구 보관되어 있다"는 증거**가 없으면 [[Invoice]]는 시장에 진열될 수 없다. 가짜 쪽지 진열 방지를 위한 1차 방어선.

#### 메타데이터 안에 뭐가 들어가나

NFT 표준에 가까운 속성 묶음(`invoice-ipfs.service.ts:154`):

- `faceValue`, `dueDate`, `fractionCount` 등 기본 채권 정보
- `defaultPaymentMethod` (FIAT_SELLER / TADA / WALLET / GUARANTOR — 자세한 건 [[수익-모델]] 섹션)
- `isGuaranteed`, `coverageRatio`, `guaranteeFee` (보증인 경우)
- Seller / Payer / Guarantor의 `externalRef` 등 식별 정보

이 묶음이 IPFS에 평생 박혀, 누구든 CID만 있으면 채권의 원본 명세를 검증할 수 있다.

---

### 한 장 요약

```
┌──────────────────────────────────────────────────────────────┐
│  APR        = 단리 연이율 (APY ≠ APR, 복리 아님)              │
│  Invoice    = 토큰화된 매출채권 (DRAFT→…→SOLD_OUT, UNPAID→…) │
│  Fraction   = 조각 = NFT 도장 (소액·분산·위변조 차단)         │
│  ipfsCid    = IPFS 번호표 (온체인 등록 필수 게이트)          │
└──────────────────────────────────────────────────────────────┘
```

이 네 단어가 뼈대다. 다음 섹션부터는 이 뼈대 위에 [[등장-인물-5명]], [[가격-전략]], [[보증-구조]], [[금고-3형제]]를 올린다.

## 등장 인물 5명과 실세계 예시

[[AR-Service]] 기반 [[RWA]] 인보이스 투자 상품을 이해할 때 가장 먼저 헷갈리는 부분이 **누가 누구인지** 입니다. 특히 "판매자"와 "지급인"을 같은 사람으로 착각하면 전체 플로우가 무너집니다. 이 섹션에서는 5명의 등장 인물을 명확히 정의하고, 두 개의 실세계 시나리오로 누가 누구인지 짚어봅니다.

### 등장 인물 5명 한 줄 요약

| # | 역할 | 영문 | 한 줄 정의 | 시스템상 위치 |
|---|------|------|------------|----------------|
| 1 | **운영자** | Operator / Admin | [[Seller]]·[[Payer]]·[[Guarantor]]를 심사·등록하고 정산을 조율하는 [[MVL]] 측 주체 | `admin-service` 권한, `@TrustedController` 호출 |
| 2 | **판매자** | [[Seller]] | "받을 권리(매출채권)"를 가진 사람. 만기 전에 채권을 팔아 **현금을 미리 받는 쪽** | [[SellerRegistry]] 화이트리스트 |
| 3 | **지급인** | [[Payer]] | 판매자에게 빚을 지고 **다달이 갚아야 하는 채무자** | [[PayerRegistry]] 화이트리스트, `defaultPaymentMethod` 보유 |
| 4 | **투자자** | [[Investor]] | 채권 조각([[Fraction]])을 할인가에 미리 사두고 만기에 액면가를 받아 [[APR]] 차익을 얻는 쪽 | 등록소 없음, 프로필 + JWT |
| 5 | **보증인** | [[Guarantor]] | 지급인이 못 갚을 때 [[Guarantor Vault]] 담보로 투자자 손실을 일정 비율(`coverageRatio`) 메워주는 보험사 역할 | [[GuarantorRegistry]] + 온체인 담보 |

> [!warning] 가장 큰 함정: 판매자 ≠ 지급인
> "판매자가 다달이 갚는다"라고 이해하면 **전체 플로우가 거꾸로** 됩니다.
> - **판매자(Seller)** = 채권을 **팔아서 현금을 받는 채권자** (돈을 받음)
> - **지급인(Payer)** = 그 채권을 **갚아야 하는 채무자** (돈을 보냄, 다달이 입금)
>
> 두 사람은 **다른 인물**입니다. 한 인보이스 = 한 달치 할부 한 장이며, 다달이 갚는 사람은 **항상 지급인**입니다.

---

### 시나리오 A — TADA 기사 전기오토바이 할부

[[MVL]] 본업이 동남아 차량공유 슈퍼앱 [[TADA]]라는 점에서 가장 자연스러운 실세계 예시입니다.

```
┌──────────────────┐         ┌──────────────────┐
│  메콩모터스       │  팔다   │   소페아          │
│  (전기오토바이    │ ──────▶│  (프놈펜 TADA     │
│   딜러)          │  300만원│   기사)          │
│   = 판매자        │  12개월 │   = 지급인        │
│   (Seller)       │  할부    │   (Payer)        │
└────────┬─────────┘         └────────▲─────────┘
         │                              │
         │ 채권(받을 권리)             │ 다달이 25만원
         │ 12장의 Invoice              │ (TADA 수익에서
         │ 발행                         │  자동 차감)
         ▼                              │
┌──────────────────┐         ┌──────────────────┐
│  지훈            │  Invoice │  앙코르캐피탈     │
│  (한국 USDC      │  조각(10) │  (캄보디아       │
│   투자자)        │  280만원에│   신용보증사)    │
│   = 투자자        │  구매    │   = 보증인        │
│   (Investor)     │          │   (Guarantor)    │
└──────────────────┘          └──────────────────┘
```

**각 인물의 행동:**

| 인물 | 실세계 행동 | 시스템상 행동 |
|------|-------------|----------------|
| 메콩모터스 (Seller) | 소페아에게 오토바이를 300만원에 외상으로 팖. 즉시 현금 280만원이 필요함 | [[SellerRegistry]]에 등록 → [[Purchase Escrow]]에서 조기 정산 받음 |
| 소페아 (Payer) | TADA 기사로 일하며 매달 25만원씩 12개월 갚음. `defaultPaymentMethod=TADA` | [[PayerRegistry]] 등록, `externalRef`에 TADA 기사 ID 연결 |
| 지훈 (Investor) | 조각 10개 중 일부를 280만원어치 구매 → 12개월 뒤 300만원 회수 (≈ 20만원 차익) | [[Purchase Escrow]]에 USDC 입금, 만기 후 redeem |
| 앙코르캐피탈 (Guarantor) | 소페아가 못 갚을 경우 240만원(=300만 × 80%)까지 보전. 그 대가로 보증료 받음 | [[Guarantor Vault]]에 담보 예치, `coverageRatio=80`, `fee=5` |
| MVL (Operator) | 세 당사자를 KYC·심사한 뒤 등록, 조각화·온체인 등록 조율 | [[Admin Service]]에서 Registry 등록, [[Contract]] 생성 |

> [!example] 돈의 방향 한 줄로
> **메콩모터스 → 오토바이 → 소페아**, **소페아 → 매달 25만 → Payback Escrow → 지훈**, **지훈 → 280만 → Purchase Escrow → 메콩모터스(조기 정산)**.
> 즉 **판매자는 미리 받고**, **지급인은 다달이 갚고**, **투자자는 미리 내고 만기에 회수**합니다.

---

### 시나리오 B — 원두농장과 카페체인

B2B 무역금융 색채가 짙은 예시입니다. 동남아 SME가 흔히 겪는 **흑자도산(매출은 있는데 현금이 없음)** 구조를 그대로 보여줍니다.

```
┌──────────────────┐         ┌──────────────────┐
│  원두농장         │  팔다   │   서울커피체인    │
│  (베트남 SME)    │ ──────▶│   (체인 매장 50개)│
│   = 판매자        │  5,000만 │   = 지급인        │
│   (Seller)       │  90일 후 │   (Payer)        │
│                  │  결제 조건│                  │
└────────┬─────────┘         └────────▲─────────┘
         │                              │
         │ "90일 뒤 5,000만 받음"        │ 90일 뒤 일시불 또는
         │ 채권 1장                     │ 분할 송금
         ▼                              │
┌──────────────────┐         ┌──────────────────┐
│  글로벌 투자자    │  4,800만에│  서울커피 본사    │
│  (USDC 보유)     │  채권 구매│  자체 보증        │
│   = 투자자        │          │   = 보증인        │
│   (Investor)     │          │   (Guarantor)    │
└──────────────────┘          └──────────────────┘
```

**핵심 포인트:**

- **원두농장(Seller)** 은 거래는 성사됐지만 **90일을 못 기다리고** 운영자금이 급함 → 채권을 4,800만에 팖
- **서울커피체인(Payer)** 은 만기일에 5,000만원을 한 번에(또는 분할로) 지급
- **투자자**는 90일 뒤 5,000만 회수 → 약 200만원 차익 ([[APR]] 환산)
- **서울커피 본사가 보증인**으로 들어오는 케이스도 가능 → 자기 자회사(매장)의 채권을 본사가 보증 → 투자자 안심 → 채권이 더 비싸게 팔림

| 인물 | 실세계 정체성 | 왜 참가하나 |
|------|---------------|-------------|
| 원두농장 | 베트남 농가/협동조합 | **현금 갭** 해결 (전통 팩토링은 거절당함) |
| 서울커피체인 | 한국 대형 가맹점 본사 | 후불 결제 조건 그대로, 변화 없음 |
| 글로벌 투자자 | 스테이블코인 보유자, DeFi 사용자 | **실물 기반 수익** (DeFi 폰지 대신) |
| 서울커피 본사 (보증인) | 가맹점 본사 = 신용도 높은 모회사 | 자회사 자금조달 + 보증료 수익 |

---

### 시나리오 비교 — 판매자/지급인 구분법

> [!tip] 헷갈릴 때 자문할 3가지
> 1. **"채권을 누가 들고 있나?"** → 그 사람이 [[Seller]]. 채권을 팔아 **현금이 급한 쪽**.
> 2. **"매달(또는 만기에) 누가 갚나?"** → 그 사람이 [[Payer]]. `defaultPaymentMethod`를 가진 채무자.
> 3. **"누가 미리 돈을 내고 나중에 회수하나?"** → 그 사람이 [[Investor]].

| 질문 | 시나리오 A | 시나리오 B |
|------|------------|------------|
| 받을 권리를 가진 사람? | 메콩모터스 | 원두농장 |
| 다달이/만기에 갚는 사람? | 소페아 (TADA 기사) | 서울커피체인 |
| 미리 현금을 대는 사람? | 지훈 (한국 투자자) | 글로벌 USDC 투자자 |
| 보증인 후보? | 앙코르캐피탈 / 메콩모터스 자체 / [[MVL]] (`isInternalMvl`) | 서울커피 본사 |
| `defaultPaymentMethod` | `TADA` (수익에서 자동 차감) | `FIAT_SELLER` (계좌이체) 또는 `WALLET` |
| 할부 횟수 (`invoiceCounts`) | 12 (월 단위) | 1 (만기 일시불) 또는 3~6 |

> [!note] 한 사람이 두 역할을 겸할 수 있다
> - **판매자 = 보증인**: 메콩모터스가 자기 채권을 직접 보증 (신뢰 신호 → 채권이 비싸게 팔림)
> - **운영자 = 보증인**: [[MVL]]이 보증인이 되는 케이스 (`isInternalMvl=true`), TADA 기사를 가장 잘 알기에 위험 통제 가능
> - 그러나 **판매자 = 지급인**은 불가능: 자기가 자기에게 빚을 지는 셈이 됨

---

### 보증인은 어디서 끼는가 — 시나리오 A에 보증인 얹기

지훈이 묻습니다: *"소페아가 갑자기 TADA를 그만두면? 오토바이 사고 나면? 내 280만원은?"*

여기서 [[Guarantor]]가 거래의 **4번째 당사자**로 등장합니다.

```
        없을 때 (비보증)                  있을 때 (보증)
        ───────────────                  ───────────────
   디폴트 → 지훈이 다 떠안음        디폴트 → Guarantor Vault에서
   APR ↑ (위험 프리미엄)                     coverageRatio만큼 인출
   투자자 모이기 어려움                APR ↓ (안전 프리미엄)
                                      보증인은 fee 챙김
                                      투자자 빠르게 모임
```

**보증인 3가지 케이스 (시나리오 A 기준):**

1. **전문 보증기관** — 앙코르캐피탈이 사전에 [[Guarantor Vault]]에 USDC 담보 예치, `coverageRatio=80`, `fee=5`. 디폴트 발생 시 컨트랙트가 자동으로 Vault에서 240만원까지 끌어옴.
2. **판매자 자기보증** — 메콩모터스 = 판매자 + 보증인. "자기가 판 오토바이의 할부를 자기가 보증" → 강한 신뢰 신호, 채권 프리미엄.
3. **MVL 자기보증 (`isInternalMvl=true`)** — [[MVL]]이 보증인. TADA 기사 데이터를 직접 보유하므로 위험 평가가 가장 정확함. 보증료 수익 + TADA 수익 차감(`defaultPaymentMethod=TADA`)으로 디폴트 위험 통제.

> [!note] 보증인은 "보험사", 보증료는 "이익에 매기는 성과보수"
> 보증료는 **액면가에 매기는 게 아니라 투자자 이익에 매기는 %** 입니다. 자세한 계산식과 예시는 별도 섹션에서 다룹니다.

## 운영자가 참가자를 등록하는 이유와 RWA 수요

### 등록은 "실세계 신뢰를 시스템에 박는 작업"이다

[[AR-Service]]에서 가장 자주 오해받는 부분은 "왜 운영자가 굳이 [[Seller]] · [[Payer]] · [[Guarantor]]를 일일이 등록해야 하는가"다. 결론부터 말하면, **등록은 시스템이 신용을 판단하는 절차가 아니라, 사람이 오프체인에서 이미 판단한 신뢰를 온체인에 박아 넣는 절차**다. [[RWA]](Real-World Asset) 토큰화의 본질이 여기 있다 — 컨트랙트는 똑똑하지 않다. 컨트랙트는 "이 주소가 진짜 채권자/채무자/보증인이 맞다"는 사실을 모른다. 그래서 사람이 미리 검증하고, 그 결과를 컨트랙트가 자동으로 강제할 수 있는 형태(=화이트리스트)로 바꿔 놓는다.

> [!note] 핵심 명제
> AR-Service의 등록은 "신용 판단 → DB 기록 → 온체인 Registry 화이트리스트"라는 **3단계 신뢰 이전 작업**이다. 시스템은 신용 점수를 매기지 않는다. 사람이 매긴 점수를 컨트랙트가 검문소처럼 강제하게 만들 뿐이다.

### 3단계 신뢰 이전 (Off-chain → DB → On-chain Registry)

```
┌──────────────────────────┐   ┌──────────────────────────┐   ┌──────────────────────────┐
│ 1. 오프체인 신뢰 형성     │   │ 2. 시스템 DB 기록         │   │ 3. 온체인 Registry 등록   │
│  (사람이 판단)            │   │  (AR-Service)             │   │  (Smart Contract)         │
├──────────────────────────┤──▶├──────────────────────────┤──▶├──────────────────────────┤
│ • 신용조사 (재무제표/평판)│   │ • CreatePayerDto 검증     │   │ • SellerRegistry          │
│ • KYC/AML 실사            │   │ • externalRef 매핑        │   │ • PayerRegistry           │
│ • 법적 계약서 (PG/AR 계약)│   │ • dataProvisionConsent    │   │ • GuarantorRegistry       │
│ • TADA 기사 등록 이력 등 │   │ • walletAddress 검증      │   │ • ID 예: DMPY-00001       │
└──────────────────────────┘   └──────────────────────────┘   └──────────────────────────┘
       (사람의 판단)                  (DB row 생성)                (whitelist 등록 = 검문소 활성화)
```

이 단계 중 어느 하나라도 빠지면 거래는 성립하지 않는다. 특히 3단계의 온체인 등록이 없으면 [[Invoice]]는 LISTED 상태로 가지 못한다 — 즉, 진열대에 올라갈 자격이 없다.

### 등록 시 채워지는 핵심 필드

`CreatePayerDto`(및 Seller/Guarantor DTO)에 들어가는 필드는 단순한 메타데이터가 아니라 **법적·운영적 의미**가 박혀 있다.

| 필드 | 의미 | 왜 중요한가 |
|------|------|-------------|
| `externalRef` | 외부 시스템(예: [[TADA]] 기사 ID, 사업자등록번호)의 실제 식별자 | 온체인 익명 주소를 실명 엔티티에 연결 — 디폴트 발생 시 법적 회수의 출발점 |
| `dataProvisionConsent` | `RECEIVED` / `WAITING` / `REJECTED` | 개인정보 제공 동의(KYC/AML 법적 근거). `RECEIVED` 아니면 계약 진행 불가 |
| `registeredBy` | `SELLER` / `OPERATOR` | 누가 등록 책임을 졌는지 추적. 판매자가 자기 거래처를 등록한 건지, 운영자가 직접 검증한 건지 구분 |
| `defaultPaymentMethod` | `FIAT_SELLER` / `TADA` / `WALLET` / `GUARANTOR` | 채무자가 어떤 경로로 갚을지 명시 — [[IPFS]] NFT 메타데이터에도 박힘 |
| `walletAddress` | EVM/BTC/Tezos/TON 형식 검증 | 온체인 자금 흐름의 종점. 형식 검증으로 오타·잘못된 주소 차단 |
| `status` | `ACTIVE` / `PENDING` / `REJECTED` | 현재 거래 가능 여부. 운영자가 사후에 동결 가능 |

> [!example] 실제 등록 흐름 (Payer 예시)
> 1. 캄보디아 [[TADA]] 기사 "소페아"가 전기오토바이 12개월 할부 계약
> 2. 운영자가 소페아의 TADA 기사 등록증, 신원 확인, 운행 이력 검토 → **오프체인 신용 판단**
> 3. `externalRef = TADA-DRIVER-77231`, `dataProvisionConsent = RECEIVED`, `defaultPaymentMethod = TADA`, `walletAddress = 0x…` 로 DB 등록
> 4. 온체인 `PayerRegistry`에 `DMPY-00001` 같은 ID로 화이트리스트 추가
> 5. 이제부터 컨트랙트는 이 주소가 진짜 채무자임을 "안다"

### 등록의 5가지 이득

등록이 단순한 행정 절차가 아닌 이유 — 5가지 구체적 이득이 있다.

> [!tip] 등록 = 회원제 비밀 클럽
> 누구나 들어올 수 있는 동네 PC방이 아니다. 운영자가 신원 확인하고 명단에 올린 사람만 입장 가능한 회원제 클럽. 그래서 안에서 일어나는 거래는 "익명 누군가의 거래"가 아니라 "검증된 회원들 사이의 거래"가 된다.

**① 가짜 채권 차단** — Seller가 등록되지 않으면 [[Invoice]] 자체를 만들 수 없다. "받을 권리"가 실재하지 않는 가짜 채권을 토큰화해서 투자자를 속이는 시나리오를 원천 차단. 진짜 거래에 기반한 진짜 빚에만 투자가 가능해진다.

**② 사칭 방지 (온체인 화이트리스트 검문소)** — Registry 컨트랙트는 자동 검문소다. 화이트리스트에 없는 주소가 보증인인 척, 채무자인 척 컨트랙트를 호출해도 컨트랙트가 거부한다. 사람이 일일이 검사할 필요 없이, 컨트랙트 레벨에서 사칭이 막힌다.

**③ 규제 준수 (KYC/AML)** — `dataProvisionConsent` 필드는 단순 플래그가 아니라 법적 동의 기록이다. 금융 규제 당국이 "이 거래의 당사자가 누구인지 알았는가?"를 물었을 때 답할 수 있는 근거가 된다. 신흥국 SME 금융과 크립토 결합에서 가장 발목 잡히는 지점을 미리 정리.

**④ 책임 추적 (Accountability)** — `externalRef`로 온체인 익명 주소가 실세계 법인/개인에 매핑된다. 디폴트 발생 시 "이 지갑 주인이 누구인지 모르겠다"가 아니라, "이 지갑은 사업자등록번호 XXX의 메콩모터스에 속한다"는 식으로 법적 추심·회수 절차로 이어진다.

**⑤ 오프체인 신뢰 + 온체인 자동화 결합** — 가장 큰 그림. 신용 판단은 사람이 잘하고(맥락·정성적 판단), 자금 이동·검문·집행은 컨트랙트가 잘한다(자동·24시간·투명). 등록은 이 둘을 잇는 다리다. 사람이 한 번 판단하면, 그 결과가 영구적으로 컨트랙트의 규칙이 된다.

```
[사람의 판단]   ─등록─▶   [컨트랙트의 자동 강제]
신용·KYC·계약              매 거래마다 자동 검문
(아날로그, 1회)            (디지털, 무한반복)
```

### 왜 RWA 인보이스 시장에 수요가 있는가

등록 시스템이 이렇게 복잡하게 설계된 이유는, 그 위에서 돌아가는 [[RWA]] [[Invoice]] 시장 자체에 강력한 양면 수요가 있기 때문이다. 수요가 없는데 인프라를 만들 이유는 없다.

#### 판매자(Seller) 측 수요 — 흑자도산을 막아라

가장 큰 동기는 **"흑자도산(black-ink bankruptcy)"** 회피다. SME(중소기업)는 매출이 있어도 망한다 — 매출 대금이 60~90일 뒤에 들어오는데, 그동안 인건비·재료비·임대료는 매달 나가기 때문이다. 매출 잡힌 채로 현금이 말라 무너지는 게 흑자도산이다.

```
1월: 1억 매출 (받을 권리만 생김)
2월: 임직원 월급 + 재료비 = 3천만 지출
3월: 동일
4월: 1월 매출 입금 ─────────────▶ 그동안 현금 부족으로 흑자도산
```

**기존 해결책의 한계:**

| 해결책 | 문제점 |
|--------|--------|
| 은행 대출 | 신흥국 SME는 신용 부족으로 거절 |
| 전통 [[팩토링]] (Factoring) | 느림(수주~수개월), 비쌈(고리), 대상 한정 |
| 무역금융 | [[ADB]] 추산 **전 세계 무역금융 격차 약 2.5조 달러** — 수요는 있는데 공급이 못 따라감 |

**RWA AR의 답:** 받을 권리(매출채권)를 토큰화해서 글로벌 투자자들에게 직접 할인 판매. 빠르고(수일), 싸고(중개자 제거), 누구나(신흥국 SME도) 접근 가능.

#### 투자자(Investor) 측 수요 — 진짜 수익에 대한 갈증

크립토 시장에는 거대한 [[스테이블코인]] 풀(2026년 기준 수천억 달러)이 있고, 이 자금은 **수익처를 찾고 있다**. 그런데:

> [!warning] DeFi 폰지의 한계
> 2020~2022년의 고APR DeFi는 대부분 토큰 발행으로 토큰을 보상하는 폰지 구조였다. 실물 기반 수익이 아니라 신규 진입자가 기존 진입자에게 지불하는 구조 → [[Terra/Luna]] · [[FTX]] 붕괴로 신뢰 무너짐. 투자자들은 이제 "진짜 어디서 돈이 나오는가?"를 묻는다.

[[RWA]] AR은 그 답이다:
- **실물 기반 수익** — 실제 기업의 매출채권 할인 차익. 출처가 명확함.
- **상대적 안정성** — 변동성 큰 알트코인 대비 단기 채권은 안정적
- **짧은 만기** — 30~90일 단위로 자금 회전, 락업 짧음
- **분산 투자** — 한 채권을 [[Fraction]]으로 쪼개 100원 단위로 여러 채권 분산
- **소액 접근** — 전통 사모채권은 최소 단위가 큰데, 토큰 조각은 누구나

#### 블록체인을 끼는 이유

이 모든 게 왜 블록체인 위에서 일어나야 하는가? 5가지 이점:

| 이점 | 설명 |
|------|------|
| **국경 없음** | 캄보디아 채권 ↔ 한국 투자자, 은행 거치지 않고 직접 매칭 |
| **24시간** | 시장 마감 없음, 주말 없음 |
| **소액 가능** | [[Fraction]] 토큰화로 100원 단위 분산 투자 |
| **투명성** | 모든 거래가 온체인에 기록, 누가 얼마 갚았는지 검증 가능 |
| **중개자 제거** | 은행·증권사 수수료 없이 직접 연결 |

#### 거시적 배경 — 왜 지금인가

> [!example] 2026년 RWA 시장의 4가지 거시 동인
> 1. **스테이블코인 폭발** — USDC/USDT 시총 수천억 달러, 이 자금이 수익처를 갈구
> 2. **금융 포용** — 신흥국 SME가 글로벌 자본에 직접 접근하는 첫 통로
> 3. **규제 명확화** — 미국·EU·싱가포르 등에서 토큰화 자산 규제 프레임 정착
> 4. **기관 진입** — [[BlackRock]] · [[Franklin Templeton]] 등 전통 금융 기관이 RWA 토큰 펀드 출시 시작

#### 잔존 리스크와 본 시스템의 대응

수요가 강한 만큼 리스크도 명확하다. AR-Service의 모든 설계 요소가 이 리스크에 대한 대응이다.

| 리스크 | 본 시스템의 대응 |
|--------|------------------|
| **신용 위험** (채무자 디폴트) | [[Guarantor]] · [[Guarantor Vault]] 담보로 보전 |
| **오프체인 신뢰 의존** | 위에서 설명한 3단계 등록 + KYC + `externalRef` |
| **유동성** | [[Fraction]]화로 2차 시장 가능성 확보 |
| **규제 불확실성** | `dataProvisionConsent` · `registeredBy` 등 규제 대응 필드 내장 |

> [!note] 종합
> 등록 시스템이 복잡한 이유는 RWA 시장 수요가 강하기 때문이다. 사람들이 정말 사고 싶어 하고, 정말 팔고 싶어 하므로, 그 거래를 안전하게 만들 안전장치가 필요하다. 운영자의 참가자 등록은 그 안전장치의 첫 단추다 — 이게 없으면 위의 4가지 거시 동인이 아무리 강해도 사기와 디폴트로 무너진다.

## 투자 상품이 만들어지는 흐름 (Contract → Invoice → LISTED)

투자 상품(=[[Invoice]] 조각)이 진열대에 올라가기까지는 **0단계 등록 → 1단계 계약서 → 2단계 쪽지 생성 → 3단계 확정 → 4단계 IPFS → 5단계 온체인 → 6단계 보증 등록 → 7단계 상장**의 일직선 파이프라인을 탄다. 각 단계는 다음 단계로 넘어가기 전 **선행 조건**과 **잠금/재시도 안전장치**가 걸려 있다.

### 0단계 — 참가자 등록 (Registry)

운영자가 먼저 실세계의 신뢰를 시스템에 박아 넣는다. 이 단계가 없으면 **투자 상품 자체가 만들어질 수 없다.**

| Registry | 등록 대상 | 온체인 화이트리스트 | 예시 ID |
|---|---|---|---|
| [[SellerRegistry]] | 채권을 파는 [[Seller]] | ✅ | DMSL-00001 |
| [[PayerRegistry]] | 돈을 갚는 [[Payer]] | ✅ | DMPY-00001 |
| [[GuarantorRegistry]] | (옵션) [[Guarantor]] | ✅ | DMGR-00001 |
| (등록소 없음) | [[Investor]] | ❌ (프로필 + JWT) | — |

> [!note] 등록은 단순 DB Insert가 아니라 **온체인 REGISTER 트랜잭션**까지 포함된다. 컨트랙트의 화이트리스트에 박힌 주소만 이후의 Invoice/Guarantee 등록을 받아준다.

`CreatePayerDto` 핵심 필드: `externalRef`, `dataProvisionConsent`(KYC 동의), `registeredBy`(SELLER/OPERATOR), `defaultPaymentMethod`(FIAT_SELLER/TADA/WALLET/GUARANTOR), `walletAddress`(EVM/BTC/Tezos/TON 형식 검증), `status`(ACTIVE/PENDING/REJECTED).

---

### 1단계 — Contract 생성 (붕어빵 틀)

[[Contract]]은 **개별 Invoice가 아닌, Invoice 12장(=12개월 할부)을 한꺼번에 찍어내는 틀**이다. 한 번 잘 만들어 두면 generateInvoices 한 방으로 12장이 우르르 만들어진다.

`schema.prisma:301` 핵심 필드:

```ts
model Contract {
  sellerId                  String
  payerId                   String
  paymentAmountPerInstallment Decimal  // 매달 갚을 금액 (=한 Invoice의 faceValue)
  invoiceCounts             Int        // 할부 횟수 = 만들 Invoice 개수
  startDate                 DateTime
  paymentDateMonthly        Int        // 매달 며칠에 갚을지 (1~28)
  fractionCount             Int        // 한 Invoice를 몇 조각으로 자를지
  strategy                  PricingStrategy   // FIXED_APR / LINEAR_DECAY / STEP_DECAY
  strategyParams            Json              // {minApr, maxApr, decaySteps, ...}
  listingCurrency           String
  cutoffDays                Int               // 만기 며칠 전까지 판매할지
  status                    ContractStatus    // DRAFT → ...
  payerConfirmation         ConfirmationStatus  // WAITING/RECEIVED/REJECTED
}
```

> [!warning] Contract는 `DRAFT`로 시작하며, `payerConfirmation = RECEIVED`가 되기 전에는 다음 단계로 넘어갈 수 없다. 즉 **채무자([[Payer]])의 사전 동의** 없이는 쪽지가 만들어지지 않는다.

> [!example] 시나리오 A 적용
> 메콩모터스(Seller)가 소페아(Payer)에게 300만원짜리 전기 오토바이를 **12개월 할부**로 판다면 →
> `paymentAmountPerInstallment = 250,000`, `invoiceCounts = 12`, `fractionCount = 10`, `strategy = LINEAR_DECAY`, `strategyParams = { minApr: 6, maxApr: 12 }`.

---

### 2단계 — generateInvoices (12장 자동 생성)

`invoice-schedule-generator.service.ts`의 `generateInvoices`가 Contract를 읽어 **Invoice N개를 한 번에 인서트**한다.

```
Contract (12개월 할부)
   │  generateInvoices()
   ▼
┌──────────────────────────────────────────────────────────┐
│ Invoice #1  dueDate = startDate + 1mo   APR = maxApr     │
│ Invoice #2  dueDate = startDate + 2mo   APR = (보간)     │
│ ...                                                      │
│ Invoice #12 dueDate = startDate + 12mo  APR = minApr     │
└──────────────────────────────────────────────────────────┘
   각각 status = DRAFT, fractionCount/availableCount 동일
```

#### APR 보간 (`spreadStrategyParams`)

> [!tip] **만기까지 오래 기다리는 쪽이 APR 더 높다.** 빨리 회수되는 1번 쪽지는 maxApr, 가장 늦게 회수되는 12번 쪽지는 minApr. 이렇게 해야 투자자가 "기간 vs 수익"을 합리적으로 고를 수 있다.

| Strategy | 1번 Invoice | 6번 Invoice | 12번 Invoice |
|---|---|---|---|
| `FIXED_APR(apr=7.5)` | 7.5% | 7.5% | 7.5% |
| `LINEAR_DECAY(max=12, min=6)` | 12% | 9% | 6% |
| `STEP_DECAY(max=12, min=6, decayRate, decaySteps)` | 12% | 단계별 ↓ | 6% |

#### 전역 잠금

```ts
@RedisLock('invoice-generation')   // 전역 단일 잠금
async generateInvoices(contractId) { ... }
```

> [!warning] `@RedisLock('invoice-generation')`은 **Contract별이 아니라 전역**이다. invoiceId(100000부터 1씩 증가)가 절대 중복되지 않도록 시스템 전체에서 한 번에 한 Contract만 generate가 돌아간다.

---

### 3단계 — confirmAllInvoices (DRAFT → CONFIRMED)

운영자가 12장을 다 훑어보고 OK를 누르면 **한 번의 `$transaction`** 안에서 다음 일을 한꺼번에 한다.

```
$transaction(async tx => {
  await tx.invoice.updateMany({ contractId, status: DRAFT })
                  .set({ status: CONFIRMED })

  await tx.contract.update({ status: INVOICE_GENERATED })

  for (invoice of invoices)
    await tx.outbox.create({ type: 'IPFS_UPLOAD', invoiceId, status: PENDING })
})
```

> [!note] 12장 중 1장이라도 잘못되면 **전부 롤백된다.** 부분 확정으로 인한 inconsistent state(일부만 IPFS 업로드 큐에 들어가는 사고)를 막는다.

이 시점부터 [[Outbox]]에 `IPFS_UPLOAD` 작업이 N개 쌓이고, 워커가 비동기로 끌고 간다.

---

### 4단계 — IPFS 업로드 → ipfsCid (번호표)

[[invoice-ipfs.service.ts]]가 NFT 메타데이터 JSON(faceValue, dueDate, payer info, **defaultPaymentMethod**, …)을 만들어 [[Pinata]]에 올리고 `ipfsCid`를 받아 invoice 레코드에 박는다.

> [!warning] `ipfsCid`가 없으면 **온체인 등록이 거부된다.** 컨트랙트는 "원본 증명서가 어디 있는지(=ipfsCid)" 없는 가짜 채권을 받지 않는다.

성공 시 Outbox는 `IPFS_UPLOAD: COMPLETED`로 마감되고, 후속으로 `SC_REGISTER` 작업이 PENDING으로 들어간다.

---

### 5단계 — SC_REGISTER (온체인 도장)

워커가 OnchainTransaction 큐에서 `SC_REGISTER`를 끌어다 [[MusubiARProtocol]] 컨트랙트의 invoice 등록 함수를 호출한다. APR과 strategy는 **bps(×100)**, **enum 숫자(LINEAR_DECAY=0, FIXED_APR=1, STEP_DECAY=2)**로 변환된다(`invoice-onchain.service.ts:127`).

```
사람용 7.5%  →  on-chain bps 750
strategy=LINEAR_DECAY → 0
```

> [!warning] 실패 시 [[Outbox]] 재시도 정책: PENDING → PROCESSING → FAILED → … 최대 **5회**까지 자동 재시도. 5회 모두 실패하면 `PERMANENTLY_FAILED` + Slack 알림. 돈 관련 작업이 조용히 사라지지 않게 하는 안전장치.

이 단계가 성공하면 invoice의 `onchainStatus = CONFIRMED`, `txHash`가 박힌다.

---

### 6단계 — 분기: 보증이면 GuaranteePosition 등록, 아니면 즉시 LISTED

> [!note] **여기서 비보증 / 보증의 리스팅 경로가 갈린다.** 이게 [[isGuaranteed]] 플래그의 가장 큰 실용적 의미다.

```
                  ┌──────────── isGuaranteed = false ────────────┐
                  │                                              │
   SC_REGISTER ───┤                                              ▼
   (성공)         │                                          LISTED 🎉
                  │                                              ▲
                  └─ isGuaranteed = true ─┐                      │
                                          ▼                      │
                          GuaranteePosition 온체인 등록 ─────────┘
                          (bulk-register-guarantee-positions)
                          성공 시 Contract = GUARANTEE_REGISTERED
```

| 분기 | 트리거 위치 | LISTED 조건 |
|---|---|---|
| 비보증 | `invoice.message.controller.ts:181` | SC_REGISTER 성공 즉시 |
| 보증 | `guarantee-position.message.controller.ts:90` | **모든** invoice의 GP 등록 성공 후 |

#### GuaranteePosition (모델 요약, `schema.prisma:407`)

```ts
model GuaranteePosition {
  invoiceId      Int      @unique          // 1 invoice : 1 GP
  guarantorId    String
  coverageRatio  Decimal(5,2)              // 0~100, 손실 보전 한도(faceValue 기준 %)
  fee            Decimal(5,2)              // 0초과~100, 이익에 대한 수수료 %
  status         GuaranteePositionStatus
}
```

> [!tip] Contract 단위로 보면, 그 계약에 속한 모든 보증 invoice에 GP가 등록 완료되어야 비로소 Contract 상태가 `GUARANTEE_REGISTERED`로 넘어간다. 즉 **계약 단위의 "보증 완비" 신호**가 따로 있다.

---

### 7단계 — LISTED (진열대 등장)

```
status = LISTED
availableCount = fractionCount   (전체 조각 판매 가능)
```

투자자가 마켓에서 살 수 있는 상태가 된다. 이후 상태머신은 구매가 진행되며 흘러간다.

---

### 전체 상태 흐름 — `InvoiceSavedStatus`

```
┌──────────┐  confirmAllInvoices   ┌────────────┐  SC_REGISTER + (GP)  ┌──────────┐
│  DRAFT   │ ───────────────────▶  │ CONFIRMED  │ ──────────────────▶  │  LISTED  │
└──────────┘     ($transaction)    └────────────┘                       └────┬─────┘
                                                                            │ 첫 구매
                                                                            ▼
                                                                  ┌──────────────────┐
                                                                  │ PARTIALLY_FUNDED │
                                                                  └────┬─────────────┘
                                                                       │ 마지막 조각 판매
                                                                       ▼
                                                                  ┌──────────┐
                                                                  │ SOLD_OUT │
                                                                  └──────────┘
```

> [!note] **판매 상태(InvoiceSavedStatus)**와 **상환 상태(InvoicePaymentStatus: UNPAID → PARTIALLY_PAID/OVERDUE/MISSED → PAID/DEFAULTED)**는 직교한다. `SOLD_OUT`이어도 `UNPAID`일 수 있고, `LISTED` 단계에서는 아직 [[Payer]]가 갚을 일이 없다.

---

### 안전장치 요약 (왜 이렇게 단계가 많은가)

| 안전장치 | 위치 | 막는 사고 |
|---|---|---|
| `payerConfirmation = RECEIVED` | Contract DRAFT | 채무자 모르게 채권 생성 |
| `@RedisLock('invoice-generation')` | generateInvoices | invoiceId 중복 |
| `$transaction(confirmAllInvoices)` | 3단계 | 부분 CONFIRMED |
| `ipfsCid` 필수 체크 | 5단계 직전 | 메타데이터 없는 가짜 채권 |
| [[Outbox]] 5회 재시도 + Slack | 4·5·6단계 | 일시 장애로 인한 실종 |
| GP 전수 등록 후 LISTED | 6→7단계 | 보증 미비 상품이 진열대로 나가는 사고 |

요약: **참가자 등록(0) → 틀(1) → 12장 생성(2) → 일괄 확정(3) → 증명서(4) → 온체인 도장(5) → (보증 등록 6) → 진열(7)**. 각 칸마다 잠금·트랜잭션·재시도·필수 필드가 박혀 있어, 한 발 실수해도 그 단계에서 멈추지 다음으로 새지 않는다.

## 가격 전략 3종과 Outbox(편지함)

쪽지([[invoice]])는 한 장에 하나의 [[faceValue]](액면가)와 만기일([[dueDate]])이 박혀 있는 "한 달 뒤 1000원 줄게" 같은 약속이야. 그런데 그 쪽지를 **언제 사느냐**에 따라 투자자가 얻는 이익이 달라져야 공평해 — 만기까지 30일 남았을 때 사는 사람과 1일 남았을 때 사는 사람이 같은 가격에 사면 이상하잖아? 그래서 [[Strategy|가격 전략]]이라는 게 필요해.

> [!note] 모든 전략의 공통 원리 — 녹는 아이스크림
> 시간이 지나 만기에 가까워질수록 **할인폭이 줄어들면서 가격이 액면가에 수렴**해. 즉 `price(t) → faceValue as t → dueDate`. APR(연이율)이 같아도 남은 일수가 줄면 할인액 자체가 작아지는 거지. 이게 "왜 늦게 사면 수익이 작아지는가"의 본질이야.

### 전략 3종 비교

| 전략 | strategy enum | strategyParams | 가격 변화 모양 | 비유 |
|---|---|---|---|---|
| [[FIXED_APR]] | **1** | `{ apr }` | 평평한 단일 APR | 정찰제(전 기간 같은 %) |
| [[LINEAR_DECAY]] | **0** | `{ maxApr, minApr }` | 직선으로 감소 | 미끄럼틀 |
| [[STEP_DECAY]] | **2** | `{ maxApr, minApr, decayRate, decaySteps }` | 주기마다 뚝 떨어짐 | 계단 |

ASCII로 보면 APR(=할인율)이 시간에 따라 이렇게 움직여:

```
APR
 │           FIXED_APR
 │  ─────────────────────────
 │
 │  \         LINEAR_DECAY
 │   \
 │    \
 │     \
 │      \____
 │
 │  ──┐       STEP_DECAY
 │    └──┐
 │       └──┐
 │          └──┐
 └─────────────────────────► t (구매 시점)
   listing                  dueDate
```

`decaySteps`는 [[StepPeriod]] enum으로 `0=daily / 1=weekly / 2=bi-weekly / 3=monthly`. "몇 단위마다 한 칸씩 떨어질래?"를 정함. `decayRate`는 한 계단의 낙폭이고.

> [!example] LINEAR_DECAY 워크드 예시
> 액면가 1,000,000원, 만기 30일, `maxApr=12%`, `minApr=4%`라고 하자.
> - 리스팅 직후(t=0): APR≈12% → 할인 폭 큼 → 싸게 살 수 있음(수익↑)
> - 만기 직전(t=30): APR≈4% → 할인 폭 작음 → 거의 액면가
> 같은 쪽지인데 일찍 산 사람일수록 더 많은 이익을 보장받는 구조.

### DB % ↔ 온체인 bps 변환 (×100)

ar-service의 DB는 **사람이 읽기 쉬운 퍼센트**로 저장해 — `apr=7.5`는 7.5%야. 그런데 [[Solidity]] 컨트랙트는 소수점을 못 다루니까 **basis points(bps)** 정수로 보내야 해. 그래서 온체인 전송 직전에 **×100**을 한다.

```
DB  : apr = 7.5        ( % )
SC  : apr = 750         ( bps,  1bp = 0.01% )
변환: bps = percent * 100
```

> [!warning] 소수 오차 방지 규칙 (invoice-onchain.service.ts:127 부근)
> 모든 APR/fee 류 값은 **DB에 %로**, **온체인에 bps로** 흐름. 코드에서 `apr.times(100)` 같은 게 보이면 십중팔구 이 컨버전이야. 반대로 컨트랙트에서 받아 DB에 적을 땐 ÷100. 한쪽에서만 곱하거나 빼먹으면 100배 오류가 나니까 변환 지점을 단일 함수로 모아두는 게 안전해.

마찬가지로 **strategy enum도 문자열이 아니라 숫자**로 전송돼:

```ts
// 온체인 페이로드 기준
LINEAR_DECAY = 0
FIXED_APR    = 1
STEP_DECAY   = 2
```

`coverageRatio`, `fee` 같은 보증 관련 % 값들도 같은 ×100 룰을 따른다고 보면 됨.

---

### Outbox(편지함) — 돈 작업이 조용히 사라지지 않게 하는 안전장치

마법 공책([[블록체인]])은 가끔 삐친다 — RPC가 죽고, gas가 튀고, nonce가 꼬여. 그런데 "쪽지 등록" 같은 작업이 한 번 실패했다고 사라져버리면 투자 상품 자체가 나오질 않아. 그래서 ar-service는 **온체인에 보낼 모든 작업을 먼저 DB에 적어두고, 성공할 때까지 재시도**하는 [[Outbox 패턴]]을 쓴다. 그 테이블 이름이 [[OnchainTransaction]].

```
[Service]                [DB: OnchainTransaction]              [Chain]
   │                            │                                 │
   │  1) insert(PENDING)        │                                 │
   ├───────────────────────────►│                                 │
   │                            │                                 │
   │  2) worker picks up        │                                 │
   │     markProcessing()       │                                 │
   │     PENDING → PROCESSING   │                                 │
   │◄───────────────────────────┤                                 │
   │                            │                                 │
   │  3) submit tx                                                │
   ├──────────────────────────────────────────────────────────────►│
   │                            │                                 │
   │  4-A) success → COMPLETED  │                                 │
   │  4-B) fail    → FAILED (retryCount++)                        │
   │  4-C) fail x5 → PERMANENTLY_FAILED ─► Slack 알림              │
```

### 상태 머신

| 상태 | 뜻 | 다음 |
|---|---|---|
| `PENDING` | 대기. 워커가 줍기 직전 | → `PROCESSING` |
| `PROCESSING` | 워커가 "찜"한 상태. **중복 실행 방지** | → `COMPLETED` / `FAILED` |
| `COMPLETED` | 온체인 성공, `txHash` 박힘 | (끝) |
| `FAILED` | 일시 실패. retryCount++ 하고 다시 `PENDING`으로 | → `PENDING` (재시도) |
| `PERMANENTLY_FAILED` | 5회 실패. 사람 호출 필요 | (Slack 알림) |

> [!tip] `markProcessing()`의 역할
> `PENDING → PROCESSING` 전이를 **원자적으로** 수행해서 같은 작업을 두 워커가 동시에 집어가는 사고를 막아. 여기서 한 번 잡히면 그 워커가 죽지 않는 한 다른 워커는 못 건드림. 사실상 작업 단위 락이지.

### 배치 실패 시: 단건 분리 재시도 (isRevert)

여러 건을 한 트랜잭션에 묶어 보내다가 그 중 하나가 revert해서 통째로 깨지는 경우가 있어. 이때 배치 안의 멀쩡한 건까지 같이 죽이면 손해니까, **`resetToPending`으로 각각을 단건으로 분리해서 다시 PENDING으로 돌려놓고 재시도**해. 이때 중요한 건 — **retryCount를 깎지 않는다.** 배치 실패는 그 건의 잘못이 아닐 수도 있으니까 페널티를 추가로 매기지 않는 거야.

```
[Batch tx]  →  revert (한 건 때문)
       │
       ▼
  isRevert 감지
       │
       ▼
  resetToPending(각 항목)
       │  - status: PROCESSING → PENDING
       │  - retryCount: 그대로 유지
       ▼
  단건으로 재시도 → 진짜 문제 있는 놈만 격리됨
```

### 어디에 쓰이나

쪽지 등록(`SC_REGISTER`), IPFS 업로드 후 온체인 메타 박기, [[GuaranteePosition]] 등록, payback sync(`paybackUsdc/paybackFiat`), `initiateRedeem` 등 **돈/소유권에 영향을 주는 온체인 호출은 거의 다 Outbox를 통과**한다. 즉 ar-service가 "할 일"을 기억하는 방식 자체가 outbox야.

> [!warning] 왜 이게 중요한가
> 사용자 입장에서 "투자했는데 쪽지가 안 올라옴", "갚았는데 redeem이 안 됨" 같은 사고는 곧바로 자금 사고로 이어져. Outbox가 없으면 RPC 한 번 끊겼을 때 작업이 증발해버려서 운영자가 손으로 복구해야 함. 5번까지 자동 재시도 → 6번째는 사람 호출(`PERMANENTLY_FAILED` + Slack)이라는 안전망이 있어야 24/7 무인 운영이 가능해.

> [!note] 한 줄 요약
> **가격 전략**은 "언제 사느냐"에 따라 공정한 할인을 계산하는 룰, **Outbox**는 "그 결과를 온체인에 반영하는 동안 절대 잃어버리지 않는다"는 약속. 둘 다 [[ar-service]]가 직접 돈을 만지지 않는데도 시스템이 견고하게 굴러가게 해주는 핵심 장치야.

## 보증 vs 비보증의 차이와 보증료 계산

[[AR-Service]]의 모든 [[Invoice]]는 발행 시점에 `isGuaranteed` 플래그가 박힙니다. 이 한 개의 boolean이 **위험 구조**, **APR 수준**, **리스팅 경로**, **마켓 노출 필드**, 심지어 [[Contract]] 단계의 상태 전이까지 전부 분기시킵니다.

### 1) 한 줄 정의

> [!note] isGuaranteed의 의미
> [[Payer]]가 만기에 못 갚을 때 투자자 손실을 메워줄 제3자([[Guarantor]])가 있느냐 없느냐. 있으면 `true`, 없으면 `false`.

### 2) 위험·APR 트레이드오프

| 구분 | 위험 부담 | 투자자 APR | 판매 속도 | 판매자 측 비용 |
|---|---|---|---|---|
| **비보증** (`isGuaranteed=false`) | 투자자가 손실 전부 떠안음 | **높음** (위험 프리미엄) | 느릴 수 있음 | 보증료 없음 |
| **보증** (`isGuaranteed=true`) | [[Guarantor]]가 `coverageRatio`만큼 보전 | **낮음** | 빠름 | `guaranteeFee` 부담 |

직관: 보증은 일종의 보험이다. 안전한 만큼 투자자가 받는 이자(APR)는 깎이고, 그 깎인 부분 일부가 보증인에게 흘러간다.

### 3) 리스팅 경로 차이 — 코드 레벨 분기

같은 `confirmAllInvoices` → IPFS → [[SC Register]] 파이프라인을 타지만, 마지막 한 칸이 다릅니다.

```
[비보증 경로]
DRAFT → CONFIRMED → IPFS_UPLOAD → SC_REGISTER ──► LISTED ✅
                                  (invoice.message.controller.ts:181)

[보증 경로]
DRAFT → CONFIRMED → IPFS_UPLOAD → SC_REGISTER → (대기)
                                                  │
                                                  ▼
                                  GuaranteePosition 온체인 등록 성공
                                  (guarantee-position.message.controller.ts:90)
                                                  │
                                                  ▼
                                                LISTED ✅
                                                  │
                                                  ▼
                          모든 보증 invoice GP 등록 완료 시
                          Contract.status = GUARANTEE_REGISTERED
```

> [!warning] 보증 invoice는 GP 등록 전까지는 절대 LISTED가 되지 않는다
> 즉 [[Investor]] 진열대에도 안 뜨고 구매도 불가능합니다. 보증인 [[Guarantor Vault]] 담보가 준비되고, `GuaranteePosition`이 온체인 Registry에 박혀야 비로소 시장에 풀립니다.

### 4) GuaranteePosition 모델

`schema.prisma:407` 기준 (invoiceId가 `@unique` — invoice 1장당 보증은 한 건만):

```prisma
model GuaranteePosition {
  id            String
  invoiceId     Int      @unique
  guarantorId   String
  coverageRatio Decimal  @db.Decimal(5, 2)  // 손실 보전 한도 %
  fee           Decimal  @db.Decimal(5, 2)  // 보증료 % (이익 기준)
  status        GuaranteePositionStatus
  ...
}
```

핵심 두 필드는 자주 헷갈리는 한 쌍입니다. **반드시 구분해서 기억**해야 합니다.

> [!tip] fee vs coverageRatio — 같은 % 두 개의 정체가 완전히 다르다
> - **`coverageRatio`** = **손실 보전 한도** %, 기준은 **액면가(faceValue)**
>   - 예: faceValue 300만원, coverageRatio 80% → 디폴트 시 최대 **240만원**까지 [[Guarantor Vault]]에서 끌어와 메움
> - **`fee`** = **보증인이 버는 수수료** %, 기준은 **투자자 이익(totalProfit)**
>   - 예: 이익 20만원, fee 5% → 보증료 **1만원**
>   - **투자자가 돈을 벌었을 때만** 발생 (이익 ≤ 0이면 0)

### 5) 보증인 3가지 케이스 — 누가 보증을 서나

> [!example] 시나리오: [[TADA]] 기사 소페아의 300만원 12개월 전기오토바이 할부
> 판매자=메콩모터스, 지급인=소페아, 투자자=지훈. 여기에 들어올 수 있는 [[Guarantor]] 패턴 세 가지.

**케이스 1 — 전문 보증기관 (제3자 보증)**
- 예: 앙코르캐피탈(캄보디아 신용보증사)
- 미리 [[Guarantor Vault]]에 USDC/MVL 담보 예치
- `coverageRatio=80%` 보장, `fee=5%`
- 디폴트 시 Vault에서 최대 240만원 메움 → 그 대가로 매 딜마다 보증료 수령
- 가장 일반적인 보험 모델

**케이스 2 — 판매자 자체 보증 (Self-guarantee)**
- 판매자(메콩모터스)가 동시에 보증인 역할
- "내가 판 채권은 내가 책임진다"는 **신뢰 신호** → 투자자 모집 빨라짐 → 채권을 더 비싸게 팔 수 있음
- 자기 채권에 대한 자신감을 시장에 보여주는 마케팅 도구이기도 함

**케이스 3 — MVL 내부 보증 (`isInternalMvl=true`)**
- [[MVL]] 자체가 보증인으로 들어옴
- [[TADA]] 생태계 기사에 대해 **MVL이 가장 잘 안다** (운행/수익/평판 데이터 보유)
- `defaultPaymentMethod=TADA`라면 기사 수익에서 자동 차감되므로 회수 리스크도 통제 가능
- 1석 3조: 보증료 수익 + 본업(TADA) 매출 ↑ + 자본 유치

### 6) guaranteeFee 계산 공식

`purchase-fee.service.ts:75` 기준:

```
guaranteeFee = totalProfit × (fee% / 100)        // totalProfit > 0 일 때만
            = 0                                  // totalProfit ≤ 0
```

여기서:
```
totalProfit = faceValue - purchasePrice   (투자자가 산 가격 대비 액면가)
```

> [!warning] 액면가가 아니라 **이익**에 대한 % 다
> 직관과 다르게 보증료는 faceValue × fee가 **아닙니다**. 투자자가 실제로 번 차익(`totalProfit`)에 대해서만 떼어갑니다. 그래서 시장 가격이 액면가에 거의 붙은 만기 직전 거래에서는 보증료가 거의 0에 수렴합니다.

`fee` 허용 범위 (`bulk-register-guarantee-positions.usecase.ts:414`):
- `0 < fee ≤ 100` (%)
- 소수점 2자리 (Decimal(5,2))
- **딜마다 협상값** — 고정 기본값 없음

### 7) 구체 숫자 예시 — 액면가 300만원, 산 가격 280만원

```
faceValue         = 3,000,000원
purchasePrice     = 2,800,000원
totalProfit       =   200,000원   ← 보증료 계산 기준
```

`fee%`를 바꿔가며 보면:

| fee % | guaranteeFee 공식 | guaranteeFee |
|---|---|---|
| 3% | 200,000 × 3% | **6,000원** |
| **5%** | **200,000 × 5%** | **10,000원** |
| 10% | 200,000 × 10% | 20,000원 |
| 20% | 200,000 × 20% | 40,000원 |

전체 정산을 끝까지 따라가 보면 (`platformFee`는 현재 0%):

```
순회수(net redeem) = faceValue − guaranteeFee − platformFee
                  = 3,000,000 − 10,000 − 0
                  = 2,990,000원

투자자 수익       = 순회수 − purchasePrice
                  = 2,990,000 − 2,800,000
                  = 190,000원   ← expectedReturn
```

즉 보증이 끼면 투자자 수익이 20만원에서 **19만원으로 1만원 줄고**, 그 1만원이 [[Guarantor]] 몫이 됩니다. 1만원의 대가로 디폴트 시 240만원까지 메워주는 보험을 산 셈입니다.

### 8) 마켓 노출 필드의 차이

마지막으로 진열대(API 응답)에서도 차이가 납니다.

| 필드 | 비보증 | 보증 |
|---|---|---|
| `isGuaranteed` | `false` | `true` |
| `coverageRatio` | `null` | 예: `80.00` |
| `guaranteeFee` (예상) | `null` | 계산값 노출 |
| `guarantor` 정보 | `null` | 보증인 식별자/이름 노출 |

이 메타데이터를 보고 투자자는 같은 [[APR]]이라도 "보증 끼고 안전한 5%냐, 무보증 위험한 5%냐"를 구분해 결정합니다.

## 담보 금고(Guarantor Vault) 심층

[[Guarantor Vault]]는 [[보증인(Guarantor)]]이 "디폴트 시 메워주겠다"는 약속을 **온체인 담보**로 미리 잠가두는 금고다. [[ar-service]]는 이 금고를 **직접 운용하지 않는다** — 돈은 컨트랙트가 들고 있고, ar-service는 잔액을 추적하고, 담보가 충분한지 감시하고, 자금 이동이 수상하면 알람을 울리는 **회계·감시 역할**만 한다.

### 1) Vault가 존재하는 이유

> [!note] 핵심 질문 — "보증한다"는 말은 약속만으로 충분한가?
> 아니다. 보증인이 도망가거나 잔고가 비면 [[투자자(Investor)]]는 그대로 손실을 떠안는다. 그래서 [[보증(Guaranteed)]] invoice는 **GuaranteePosition 등록 전에** 보증인이 USDC/MVL을 Vault에 예치해야 하고, ar-service는 그 액수가 [[liability]]를 충분히 덮는지를 **만기 7일 전부터** 미리 확인한다.

```
보증인 약속 ─┐
            ├─► Vault (USDC/MVL 잠금) ─► 디폴트 시 컨트랙트가 끌어옴
ar-service ─┘   감시·기록만, 직접 운용 X
```

| 토큰    | 평가 방식                                    | 비고                                  |
| ------- | -------------------------------------------- | ------------------------------------- |
| USDC    | 1:1 그대로 USD                               | 변동성 없음                           |
| BASE_MVL / MVL | 온체인 `totalUSDValue(bytes32 guarantorId)` 호출 | 변동성 때문에 ar-service는 환산 안 함 |

> [!tip] MVL→USD 환산은 항상 [[guarantor-vault-contract.service.ts]]:30의 `vaultContract.read.totalUSDValue()`로 처리한다. USDC 6 decimals로 반환. ar-service는 원시 수량(`Decimal(30,18)`)만 DB에 저장하고, 가격 판단은 컨트랙트의 실시간 평가에 100% 의존한다.

### 2) 이중 장부 — Transaction(이력) + Balance(스냅샷)

```
┌────────────────────────────────────────┐  ┌────────────────────────────────┐
│ GuarantorVaultTransaction (이력 원장) │  │ GuarantorVaultBalance (잔고)   │
│  txHash @unique                        │  │  @@unique[guarantorId,token,   │
│  type: DEPOSIT / WITHDRAWAL            │  │           network]             │
│  amount Decimal(30,18)                 │  │  amount Decimal(30,18)         │
│  status: PENDING → CONFIRMED           │  │  (입출금 확정 시 +/- 갱신)     │
└────────────────────────────────────────┘  └────────────────────────────────┘
                       │                                      ▲
                       └──────────── confirm 시 $transaction ─┘
```

> [!example] 왜 두 개로 나누나?
> - **Transaction**은 "언제·어떤 tx로·얼마가" 들어왔는지 *변경 불가능한 감사 추적*.
> - **Balance**는 "지금 이 보증인이 USDC 몇 개, MVL 몇 개를 들고 있나"를 빠르게 조회하기 위한 *집계 캐시*.
> 둘을 한 `$transaction` 안에서 동시에 갱신해서 **이력과 잔고가 한 순간도 어긋나지 않게** 한다.

### 3) 입출금 확정 경로 A vs B

> [!note] Vault 입금은 두 가지 방식으로 등록된다 — 사람이 미리 적어놓고 컨트랙트 이벤트로 확정되거나, 컨트랙트 이벤트가 먼저 와서 자동으로 매칭되거나.

**경로 A — recordDeposit (선기록 후 확정)**

```
1. 관리자(admin-service) → recordDeposit(guarantorId, txHash, amount)
2. GuarantorVaultTransaction row 생성 (status=PENDING)
3. watcher-worker가 체인에서 해당 tx 발견 → TRANSACTION_CREATED_V2 이벤트
4. ar-service의 confirm-vault-transaction.usecase 가 받아서
   PENDING → CONFIRMED + Balance += amount  (한 $transaction 안에서)
```

**경로 B — autoConfirmVaultTx (자동 감지)**

```
1. (관리자 선기록 없음)
2. watcher-worker가 Vault 컨트랙트로/에서의 transfer 감지
3. transfer.from 또는 transfer.to 의 주소로 registered guarantor 검색 → 매칭
4. Transaction row를 CONFIRMED 상태로 즉시 생성 + Balance 갱신
```

> [!tip] 둘이 공존하는 이유 — 운영자가 선등록을 잊거나, 보증인이 자기 지갑에서 직접 송금해도 자동 경로 B가 받아준다. 반대로 선등록이 있어 amount가 명확한 경우는 경로 A가 ID/메타데이터를 더 풍부하게 남긴다.

### 4) 온체인 금액 우선 + patchConfirmedTx 보정

> [!warning] 사람이 입력한 `amount`와 체인의 실제 amount가 다를 수 있다.
> 관리자가 "10,000 USDC 들어올 예정"이라 적어놨는데, 실제로는 9,998이 들어온다거나 수수료 처리로 차이가 생긴다. 어느 쪽을 진실로 보는가?

[[confirm-vault-transaction.usecase.ts]]:124의 결정:

```
온체인 금액(transfer event의 raw value) ALWAYS WINS

if (pending.amount !== onchain.amount) {
    patchConfirmedTx(txHash, onchain.amount)   // Transaction.amount 역산 보정
    Balance += onchain.amount                  // Balance도 온체인 기준으로
}
// 위 두 줄은 동일 prisma.$transaction 으로 원자적
```

| 입력원         | 신뢰도        | 처리                                |
| -------------- | ------------- | ----------------------------------- |
| 관리자 선기록  | 참고용        | onchain과 다르면 덮어씀             |
| watcher 이벤트 | **단일 진실** | Balance / Transaction 모두 이걸로   |

### 5) 담보 충분성 7일 선제 감시 크론

> [!example] [[CRON_AR_SERVICE_CHECK_GUARANTOR_COLLATERAL]] — 매일 도는 검사 선생님
> [[check-guarantor-collateral.usecase.ts]]가 매일 실행되며, **오늘부터 7일 안에 만기가 도래하는 보증 invoice**를 모두 모아 보증인별 책임 총액(liability)을 계산하고, Vault의 totalUSDValue와 비교한다.

```
for each active guarantor:
    liability = Σ (invoice.faceValue × invoice.coverageRatio / 100)
                where invoice.dueDate within [today, today + 7d]
                and  invoice.guarantorId == this guarantor
                and  invoice.paymentStatus != PAID
    vaultValue = vaultContract.read.totalUSDValue(guarantorId)   # USDC 6 decimals

    if vaultValue < liability:
        send email warning  (관리자 + 보증인)
        # 7일 전부터 알리니까 보충할 시간이 있다
```

| 항목                | 수식                                   |
| ------------------- | -------------------------------------- |
| 한 invoice의 책임   | `faceValue × coverageRatio / 100`      |
| 보증인의 총 liability | 위를 7일 내 만기 invoice 전부 합산    |
| Vault 평가          | `totalUSDValue(guarantorId)` 온체인 read |
| 경보 조건           | `vaultValue < liability`               |

> [!tip] 7일이라는 윈도우는 "발견 → 보증인 통지 → 추가 예치 → 체인 컨펌"까지 영업일 기준으로 여유를 주기 위한 [[lead time]]이다.

### 6) MonitorVaultMovementUc — 도둑 감시

> [!note] [[MonitorVaultMovementUc]]는 ar-service가 관리하는 [[금고 3형제]] 전체의 USDC 이동을 감시한다.

```
감시 대상 (infra.config.ts:330, monitored:true):
  - Guarantor Vault
  - Payback Escrow
  - Purchase Escrow
   (AR Protocol 컨트랙트는 monitored:false — 프로토콜 내부 이동이라 시그널 노이즈)

대상 토큰: USDC only
   (MVL은 가격 변동 때문에 보안 시그널로 부적합 — 정상 거래도 큰 amount로 보일 수 있음)
```

상대방 분류 — transfer의 counterparty(거래 상대 지갑)를 셋 중 하나로 라벨링:

| 분류              | 매칭 기준                                | 처리                       |
| ----------------- | ---------------------------------------- | -------------------------- |
| **정상 (known)**  | 등록된 guarantor 지갑 / investor 지갑    | 로그만, 알림 없음          |
| **internal**      | AR 프로토콜 컨트랙트 주소 (3형제 + AR Protocol) | 로그만, 알림 없음          |
| **SUSPICIOUS**    | 위 둘 어디에도 매칭 안 되는 지갑          | `@channel` Slack 알림 발사 |

> [!warning] SUSPICIOUS 알림은 **`@channel` 멘션**으로 운영진 전원을 깨운다. 정상이라면 등록 누락이거나 화이트리스트 추가가 필요하다는 신호, 아니라면 진짜 자금 유출일 수 있다.

### 7) 그래서 Vault는 왜 부족해지나?

> [!warning] 담보가 충분히 잠겨 있는데도 7일 크론이 부족 경보를 울리는 시나리오는 셋이다.

**원인 1 — MVL 가격 폭락**

```
T-30d: 1 MVL = $0.10, 보증인 100,000 MVL = $10,000 (충분)
T-1d:  1 MVL = $0.03, 보증인 100,000 MVL = $3,000  (부족!)
       ↑ ar-service는 totalUSDValue 호출로 실시간 평가 → 즉시 감지
```

**원인 2 — 공유풀 구조 (한 보증인이 여러 invoice 보증)**

```
보증인 A의 Vault: $50,000 USDC

  Invoice #1 (만기 D+3, liability $20,000) ┐
  Invoice #2 (만기 D+5, liability $20,000) ├─ 총 책임 $70,000
  Invoice #3 (만기 D+6, liability $30,000) ┘   > Vault $50,000 → 부족
```

Vault는 **글로벌 풀**이지 invoice별로 칸막이가 없다. 새 GuaranteePosition을 등록할 때마다 누적 책임이 늘어난다.

**원인 3 — 보증인이 출금**

```
정상 보증 진행 중 → 보증인이 부분 출금 (WITHDRAWAL 트랜잭션)
  → Balance 감소 → 다른 invoice들의 책임을 못 덮을 수 있음
```

> [!tip] 세 원인의 공통점: **liability는 잠겨있지만 vaultValue는 변동한다**. 그래서 ar-service는 잔액 절대값이 아니라 **liability vs vaultValue 차이**를 매일 본다. 7일 선제 윈도우가 있기에 보증인은 이 차이를 메울 영업일을 확보한다.

## 갚기(Repayment)와 디폴트 시 보상 집행

이 단계는 [[ar-service]]가 "직접 돈을 만지지 않는다"는 원칙이 가장 선명하게 드러나는 구간이다. 갚는 일은 4명이 분업하고, 디폴트가 발생하면 보상 집행은 전적으로 [[온체인]] 컨트랙트가 처리한다.

### 1. 갚기(Repayment) — 4명 분업

```
┌─────────────────┐   실제 송금        ┌──────────────────────┐
│ ① Payer         │ ───────────────▶  │ 은행 계좌 / Payback   │
│ (채무자)         │   (off-chain or   │ Escrow (on-chain)    │
└─────────────────┘    on-chain)      └──────────────────────┘
        │                                       ▲
        │  "보냈어요"                            │  watcher가 본 사실
        ▼                                       │
┌─────────────────┐   @TrustedController        │
│ ② Admin         │ ─────────────────────────┐ │
│ (admin-service) │   PaymentEvent 기록       │ │
└─────────────────┘                          │ │
        │                                    ▼ ▼
        ▼                            ┌──────────────────────┐
┌─────────────────┐   aggregator     │ ③ ar-service         │
│ DB              │ ◀───────────────│  (집계 + sync)        │
│ PaymentEvent    │                  └──────────────────────┘
└─────────────────┘                              │
                                                 │ paybackUsdc / paybackFiat
                                                 │ delta(증가분)만
                                                 ▼
                                       ┌──────────────────────┐
                                       │ ④ AR Protocol        │
                                       │  Contract            │
                                       │  (Payback Escrow 보관)│
                                       └──────────────────────┘
```

| 번호 | 주체 | 하는 일 | 비고 |
|---|---|---|---|
| ① | [[Payer]] | 실제로 돈을 보냄 | 현금이면 은행 계좌, USDC면 [[Payback Escrow]] |
| ② | [[Admin]] (admin-service) | `PaymentEvent` 수기 기록 | `@TrustedController`, channel = `ADMIN_MANUAL` |
| ③ | [[ar-service]] | 집계 후 [[온체인 sync]] | `paybackUsdc` / `paybackFiat`, delta만 |
| ④ | [[AR Protocol Contract]] | 상환금 보관 + redeem 시 지급 | Payback Escrow 잔액으로 |

> [!note] 핵심
> [[ar-service]]는 **공책 비서**다. 돈 흐름의 진실은 ① Payer가 송금한 사실과 ④ 컨트랙트의 Escrow 잔액에 있고, ②③는 그 둘을 *기록·동기화*만 한다.

#### Admin이 기록할 수 있는 PaymentEvent 타입

- `PAID` — 정상 입금
- `MISSED` — 만기 지나도 미납
- `DEFAULTED` — 회수 불가능으로 판정

> [!warning] `OVERDUE`는 수기 기록 대상이 아니다
> `OVERDUE`는 ar-service의 크론 잡이 `dueDate`와 현재 시각을 비교해 **자동 산출**한다. Admin이 직접 박지 않는다.

기록 경로:

- 단건: `recordManualPaymentEvent(...)`
- 일괄: `uploadPaymentEventsCsv(...)` — 통장 사본을 CSV로 떨궈서 한 번에 옮겨적는 형태

#### ar-service의 집계 + sync (delta-only)

`invoice-payment-aggregator.service.ts` 가 invoice별 `PaymentEvent`를 모아 `totalPaid` / `paymentStatus`를 산출한다. 그 뒤 `payment-sync-onchain.service.ts` 가 변경분만 컨트랙트로 흘려보낸다.

```ts
// 의사 코드
const syncableTotal = aggregateForSync(invoiceId)   // 누적 동기화 대상 금액
const delta = syncableTotal - lastSyncedAmount      // 직전 sync 이후 증가분

if (delta < 0n) throw new Error('negative delta — payback cannot decrease')
if (delta === 0n) return                            // 보낼 게 없음

await arProtocol.paybackUsdc(invoiceId, delta)      // 혹은 paybackFiat
```

> [!warning] 음수 delta 금지
> 컨트랙트의 `payback`은 **더하기만** 가능하다. 회계상 차감 보정이 필요한 상황에서도 ar-service는 음수 sync를 보내지 않으며, sync 직전 가드에서 즉시 에러를 던진다. 모든 보정은 컨트랙트가 인지하기 *전*에 DB 레벨에서 끝나야 한다.

> [!tip] @RedisLock으로 invoice당 직렬화
> aggregator는 invoice 단위 `@RedisLock`을 걸어 같은 invoice의 동시 집계·sync를 막는다. 대량 CSV 업로드 + 자동 크론이 겹쳐도 paybackUsdc 호출은 invoice별로 한 줄로 선다.

### 2. 디폴트 판정과 initiateRedeem

#### 2-1. 디폴트 판정 — `hasDefault` 최우선

`deriveStatus()` 의 우선순위는 단순하다.

```
hasDefault → DEFAULTED  (★ 최우선, 다른 어떤 신호보다 위)
fullyPaid  → PAID
partiallyPaid → PARTIALLY_PAID
overdue    → OVERDUE / MISSED
else       → UNPAID
```

> [!note] DEFAULTED는 한 번 박히면 끝
> Admin이 `DEFAULTED` PaymentEvent를 하나라도 박으면 그 invoice의 paymentStatus는 다른 어떤 PAID 신호로도 뒤집히지 않는다. 보증 집행 트리거이기 때문에 보수적으로 잠근다.

#### 2-2. 인출 창구 열림 조건 — 만기 도래만 본다

`findMaturedAwaitingInitiateRedeem` (invoice.repository.ts:98) 조건:

| 필드 | 조건 |
|---|---|
| `onchainStatus` | `CONFIRMED` |
| `dueDate` | `<= 오늘 + 1일` |
| `status` | `PARTIALLY_FUNDED` 또는 `SOLD_OUT` |

> [!warning] paymentStatus = PAID 조건이 없다
> 창구는 **만기**로 열리지, **돈이 다 들어왔는지로** 열리지 않는다. 디폴트라 한 푼도 안 들어왔어도 만기만 지나면 `initiateRedeem`이 호출된다. 투자자가 실제로 받는 금액은 *그때 Escrow와 Vault에 들어 있는 만큼*이다.

#### 2-3. initiateRedeem 호출

`process-initiate-redeem.usecase.ts`가 위 조건을 만족하는 invoice를 골라 컨트랙트의 `arProtocolContract.initiateRedeem(invoiceId)`를 호출한다. 이때부터 투자자는 자기 조각을 redeem할 수 있다.

### 3. FractionsRedeemed 이벤트 — 분배의 진실은 여기

투자자가 redeem 트랜잭션을 보내면 컨트랙트가 `FractionsRedeemed` 이벤트를 발화하고, [[watcher-worker]]가 이를 잡아 ar-service에 전달한다. 디코드된 페이로드:

| 필드 | 의미 |
|---|---|
| `redemptionAmountUsdc` | 투자자가 실제 받은 금액 (USDC) |
| `protocolFeeUsdc` | MVL/프로토콜이 떼간 수수료 |
| `guarantorFeeUsdc` | 보증인이 가져간 보증료 (있는 경우) |

> [!note] guarantorFee가 0이 아니면 보증인 정산이 포함된 것
> ar-service는 이 셋을 그대로 받아 적기만 한다. "보증인에게 얼마 보내라"는 명령은 ar-service 어디에도 없다 — 컨트랙트가 단일 트랜잭션 안에서 투자자/프로토콜/보증인 셋을 동시에 정산한다.

redeem 결과에 따라 상태가 갈린다:

| 케이스 | invoice 상태 |
|---|---|
| 액면가 전액 회수 | `REDEEMED` |
| 부분 회수 (디폴트 + 보증 일부) | `PARTIAL_REDEEM` |

### 4. Payback Escrow 부족 → Vault에서 coverageRatio만큼

자금 흐름의 우선순위는 컨트랙트가 결정한다:

```
redeem 요청
    │
    ▼
┌──────────────────────────────────┐
│ Payback Escrow 잔액 충분?         │
└──────────────────────────────────┘
    │ Yes                       │ No
    ▼                            ▼
[Payback Escrow에서 지급]   ┌──────────────────────────────────┐
                            │ 보증 있음 (isGuaranteed)?         │
                            └──────────────────────────────────┘
                                │ Yes                   │ No
                                ▼                        ▼
                    [Guarantor Vault에서        [부족분 = 투자자 손실]
                     coverageRatio 한도로 보충]   (비보증의 본질)
```

> [!example] 디폴트 보상 워크스루
> - faceValue 300만, coverageRatio 80%, fee 5%
> - 만기 시 Payback Escrow 잔액: 100만 (Payer가 일부만 갚음)
> - 부족분 200만 → Guarantor Vault에서 최대 `300만 × 80% = 240만`까지 보충 가능 → 200만 인출
> - 투자자에게 redemptionAmount 지급, FractionsRedeemed에 guarantorFee 채워서 보증인 정산
> - invoice 상태: 전액이 들어왔으면 `REDEEMED`, 그래도 부족하면 `PARTIAL_REDEEM`

> [!warning] 비보증은 안전망 없음
> `isGuaranteed = false` 이면 Vault 인출 단계 자체가 없다. Payback Escrow에 들어온 금액만 비례로 분배되고, 나머지는 투자자가 그대로 떠안는다. 그래서 APR이 더 높았던 것.

### 5. MVL 담보의 USD 가치 평가 — 온체인 위임

Vault에는 USDC뿐 아니라 MVL도 담보로 들어간다. MVL은 변동성이 큰데, [[ar-service]]는 MVL 가격을 **직접 평가하지 않는다**.

```ts
// guarantor-vault-contract.service.ts:30
const usd = await vaultContract.read.totalUSDValue(guarantorIdAsBytes32)
// → USDC 6 decimals 단위로 반환
```

> [!note] 가격 평가 전부 온체인
> `totalUSDValue(bytes32 guarantorId)` 한 줄이면 끝. MVL → USD 환산 로직, 오라클 의존성, 라운드다운 규칙은 전부 컨트랙트 안에 박혀 있다. ar-service는 `BASE_MVL` / `MVL` 같은 토큰 식별자만 알고, DB에는 **원시 수량(raw amount)** 만 저장한다.

이렇게 위임한 이유:

- **시세 변동성 흡수**: 코인값이 폭락한 순간에도 컨트랙트가 같은 식으로 평가 → ar-service와 컨트랙트가 다른 가격을 보는 불일치가 원천 차단
- **보안 감사 단순화**: [[MonitorVaultMovementUc]]는 USDC-only 감시로 충분 (MVL 평가 책임을 안 짊)
- **선제 감시**가 가능해짐: `check-guarantor-collateral.usecase.ts` 가 매일 7일 전 만기 보증의 `faceValue × coverageRatio` (= liability) vs `totalUSDValue` 를 비교해 부족하면 이메일로 형아를 깨운다 → 디폴트 *실제로 발생하기 전에* 충전 유도

> [!tip] 정리: ar-service의 책임 경계
> - **기록**: PaymentEvent (Admin이 박은 것)
> - **집계**: aggregator로 totalPaid 계산
> - **동기화**: delta만 컨트랙트로 push (음수 금지)
> - **조율**: initiateRedeem 트리거, FractionsRedeemed 디코드해서 invoice 상태 전환
> - **위임**: 보상 계산·자금 이동·MVL→USD 평가는 모두 컨트랙트

## 온체인 금고 3형제와 redeem 창구 열림 조건

### MVL의 지갑이 아니라 "자판기"

먼저 가장 큰 오해부터 끊고 가자. [[ar-service]] 코드베이스에는 `treasury`라는 이름의 지갑이 **0건** 존재한다. MVL이 투자자 돈을 받아두는 "회사 통장" 같은 건 없다.

대신 자금은 온체인의 **스마트컨트랙트 3개**에 잠겨 있고, 각자 정해진 규칙(조건)을 만족해야만 돈이 나온다. 그래서 [[금고 3형제]]는 지갑이라기보다 **자판기**다. 동전을 넣어도 버튼이 안 눌리면 콜라가 안 나오듯, 만기·상태·서명이 안 맞으면 돈이 안 풀린다.

> [!note] 핵심 멘탈 모델
> [[ar-service]]는 돈을 직접 만지지 않는 **장부 비서**다. 진짜 돈을 들고 있는 건 [[Guarantor Vault]] / [[Purchase Escrow]] / [[Payback Escrow]] 세 개의 스마트컨트랙트다. ar-service는 이들의 입출금 이력을 DB에 그대로 베껴 적고, 룰 위반(SUSPICIOUS)이 보이면 Slack으로 호루라기를 분다.

### 금고 3형제 역할 분리표

| 금고 | 누구 돈을 담나 | 어디서 들어와 어디로 나가나 | 코드상 위치 |
|---|---|---|---|
| **[[Guarantor Vault]]** | 보증인의 **담보**(USDC + MVL) | 보증인이 미리 예치 → 디폴트 시 컨트랙트가 [[coverageRatio]]만큼 끌어가 [[Payback Escrow]]로 흘림 | `ArContractConfig` (`infra.config.ts:330`), `monitored:true` |
| **[[Purchase Escrow]]** | **투자자가 낸 구매 대금** | 투자자 → Escrow → (만기 전) 판매자 조기 정산 | `monitored:true` |
| **[[Payback Escrow]]** | 지급인이 갚는 **상환금** (USDC) | 지급인 → Escrow → 만기 후 투자자 redeem | `monitored:true` |
| (참고) AR Protocol | 자금 보관 ❌ — 규칙(자판기 두뇌)만 담당 | `initiateRedeem`, `FractionsRedeemed` 등 정산 이벤트 | `monitored:false` |

```
        ┌────────────────┐                     ┌────────────────┐
 보증인 │ Guarantor Vault│ ─(디폴트 시 인출)─▶ │                │
        └────────────────┘                     │                │
                                               │ Payback Escrow │ ──(redeem)──▶ 투자자
        ┌────────────────┐    (지급인 상환)   │                │
 지급인 ────────────────────────────────────▶ └────────────────┘
                                               
        ┌────────────────┐    (조기 정산)
 투자자 │ Purchase Escrow│ ────────────────────────────────▶ 판매자
        └────────────────┘
```

> [!tip] "그럼 자판기는 누가 감시하나?" — **watcher-worker = 체인 CCTV**
> ar-service 자체가 체인을 폴링하지 않는다. 외부의 [[watcher-worker]]가 RPC로 블록을 훑다가 위 3개 금고 주소에서 자금 이동이 보이면 **`TRANSACTION_CREATED_V2`** 이벤트를 발행한다. ar-service는 이걸 받아서:
> 1. `MonitorVaultMovementUc`로 도둑 검사 (등록 guarantor/investor=정상, AR 프로토콜 컨트랙트=internal, 알 수 없는 지갑=**SUSPICIOUS → @channel Slack**)
> 2. `autoConfirmVaultTx`로 Guarantor Vault 입출금을 `PENDING → CONFIRMED`로 확정
>
> 즉 컨트랙트가 watch하는 게 아니라, **watcher-worker가 컨트랙트를 watch**하고 → ar-service가 그 이벤트를 듣는다.

### Purchase Escrow의 잘 안 보이는 역할 — 판매자 조기 정산(seller settlement)

[[Purchase Escrow]]는 단순히 투자자 돈을 보관만 하지 않는다. **판매자가 만기 전에 현금을 받는 창구**이기도 하다. 이게 [[RWA 인보이스 투자 상품]]이 존재하는 본 목적이다 — 판매자가 60~90일 뒤가 아니라 지금 당장 현금이 필요해서 채권을 할인 매각하는 것.

```
1) 투자자가 조각을 산다       → Purchase Escrow에 돈이 찬다
2) 판매자에게 seller settlement → Purchase Escrow에서 판매자 지갑/계좌로 송금
3) (별도) 만기 후 지급인이 갚음 → Payback Escrow에 돈이 찬다
4) 투자자가 redeem            → Payback Escrow에서 투자자에게 송금
```

> [!example] Purchase Escrow vs Payback Escrow — 헷갈리지 않는 한 줄
> - **[[Purchase Escrow]]** = 투자자 돈 → 판매자 (앞쪽 정산)
> - **[[Payback Escrow]]** = 지급인 돈 → 투자자 (뒤쪽 정산)
>
> 두 금고가 분리되어 있어서 "지급인이 아직 안 갚았는데 판매자는 이미 현금을 받음" 같은 시간차 거래가 가능하다.

### redeem 창구가 열리는 진짜 조건 — `findMaturedAwaitingInitiateRedeem`

여기서 자주 틀리는 멘탈 모델: "지급인이 갚아서 `paymentStatus=PAID`가 되면 투자자가 인출할 수 있다." **틀렸다.** redeem 창구는 **입금 여부가 아니라 만기 도래로** 열린다.

코드(`invoice.repository.ts:98` — `findMaturedAwaitingInitiateRedeem`)는 다음 세 조건을 AND로 본다:

```ts
// 의사 코드
WHERE onchainStatus = CONFIRMED
  AND dueDate <= today + 1day      // 만기가 도래(또는 임박)
  AND status IN (PARTIALLY_FUNDED, SOLD_OUT)
// 주목: paymentStatus = PAID 조건은 없음!
```

| 흔한 오해 | 실제 |
|---|---|
| `paymentStatus=PAID` 되어야 redeem 가능 | ❌ 조건에 없음 |
| `paymentStatus`가 스위치 역할 | ❌ 만기가 스위치 |
| 입금 안 되면 redeem 자체가 막힘 | ❌ 창구는 열림. 단, **금고에 든 돈만큼만** 가져감 |

> [!warning] 창구 열림 ≠ 돈이 다 들어와 있음
> `findMaturedAwaitingInitiateRedeem`이 invoice를 픽업해 `arProtocolContract.initiateRedeem`을 부르면, AR Protocol 컨트랙트가 `FractionsRedeemed` 이벤트를 쏘면서 정산한다. 이때 투자자가 **실제로 받는 금액 = 그 시점 금고에 채워진 만큼**이지, 액면가가 자동으로 보장되는 게 아니다.

### 실제 받는 금액의 분기 — "금고에 든 만큼"

| 상황 | 돈이 나오는 금고 | 결과 |
|---|---|---|
| 지급인이 만기까지 다 갚음 | [[Payback Escrow]] | `REDEEMED` — 액면가 전액 정산 (수수료 차감) |
| 일부만 갚음 | [[Payback Escrow]] (부족분) | `PARTIAL_REDEEM` — 들어온 만큼만 |
| 디폴트 + **보증** invoice | [[Payback Escrow]] (지급인 분) + **[[Guarantor Vault]]** (담보에서 [[coverageRatio]]만큼) | 컨트랙트가 자동으로 Vault 자금을 끌어와 보전 |
| 디폴트 + **비보증** invoice | (없음) | 손실은 투자자가 떠안음 |

[[FractionsRedeemed]] 이벤트를 디코드하면 `redemptionAmountUsdc / protocolFeeUsdc / guarantorFeeUsdc` 필드가 나오는데, **`guarantorFeeUsdc`가 0이 아니면 그 정산에 보증인이 개입했다**는 신호다(즉 Vault에서 돈이 끌려나왔거나, 보증인 수수료가 차감되었다는 뜻).

### 흐름을 다시 한 번 — payment event는 "스위치"가 아니라 "급유"

```
[지급인이 진짜 송금] ─────────────▶ Payback Escrow에 USDC가 쌓임
       │
       ▼
[운영자가 admin-service로
 PaymentEvent 기록]      ─────────▶ ar-service가 paybackUsdc/Fiat을 컨트랙트에 sync
                                    (증가분 delta만, 음수 불가)

[만기일 도래] ───────────────────▶ findMaturedAwaitingInitiateRedeem 픽업
                                    → arProtocolContract.initiateRedeem
                                    → FractionsRedeemed 이벤트
                                    → 투자자가 redeem 가능
```

즉 [[PaymentEvent]] 기록은 "redeem 가능/불가능"을 결정하는 스위치가 아니라, **금고(특히 Payback Escrow)에 실제 돈을 채워 넣고 그 사실을 체인에 알리는 급유 행위**다. 만기는 캘린더가 결정하고, 받을 수 있는 금액은 그때 금고에 얼마나 차 있는지가 결정한다.

> [!note] 한 줄 요약
> [[ar-service]] 코드에 `treasury`는 없다. 자금은 [[Guarantor Vault]](담보) · [[Purchase Escrow]](투자자 구매대금/판매자 조기정산) · [[Payback Escrow]](지급인 상환금/투자자 redeem) **3개의 자판기 컨트랙트**에 분리 보관되고, [[watcher-worker]]가 `TRANSACTION_CREATED_V2`로 자금 이동을 중계한다. redeem 창구는 `paymentStatus=PAID`가 아니라 **만기 도래**로 열리며, 투자자가 실제로 받는 금액은 그 시점 금고에 채워진 만큼이다.

## MVL의 수익 모델과 defaultPaymentMethod 4종

이 섹션은 두 가지를 함께 다룬다.
(1) [[invoice]]에 박히는 `defaultPaymentMethod` 4종이 **무엇을 의미하고, 시스템에서 어떤 역할을 하는지**,
(2) MVL이 [[ar-service]] 위에서 **어떻게 돈을 버는지** — 코드로 확인되는 부분과 비즈니스 추론을 명확히 구분해서.

---

### 1. `defaultPaymentMethod` 4종 — IPFS NFT 속성으로만 박히는 메타데이터

먼저 가장 흔한 오해 하나부터 정리한다.

> [!warning] `defaultPaymentMethod`는 분기 로직이 아니다
> `defaultPaymentMethod`는 [[ar-service]] 코드 안에서 "이 값이면 A로, 저 값이면 B로" 같은 **자동 처리 분기를 만들지 않는다**. 코드 어디에도 `if (defaultPaymentMethod === 'TADA')` 같은 로직 분기가 없다. 이 값은 [[invoice]]가 [[IPFS]]에 올라갈 때 NFT 메타데이터 속성(`attributes`)에 박히는 **설명용 라벨**일 뿐이다 (`invoice-ipfs.service.ts:154`). 즉 투자자가 "이 채권은 채무자가 어떤 방식으로 갚기로 했는가"를 **마켓에서 눈으로 확인하는 용도**.

실제 갚는 일은 별개의 흐름으로 일어난다 — 채무자가 진짜로 돈을 보내고([[ar-service]] 바깥), 운영자가 [[admin-service]]를 통해 `PaymentEvent`를 수동 기록하고, [[ar-service]]가 그걸 집계해서 마법 공책(컨트랙트)에 [[paybackUsdc]]/[[paybackFiat]] 만큼 sync하는 흐름. `defaultPaymentMethod`는 그 흐름에 **자동으로 끼지 않는다**.

#### 4종의 의미

| 값 | 누가 갚나 | 어디로 흐르나 | 실세계 예시 |
|---|---|---|---|
| `FIAT_SELLER` | [[지급인 Payer]]가 [[판매자 Seller]]에게 **현금/계좌이체**로 갚음 | 오프체인 → 판매자가 받아서 → 판매자 재량으로 시스템 반영 | 캄보디아 카페체인이 원두농장에 매달 송금 |
| `TADA` | [[지급인 Payer]]의 [[TADA]] 앱 수익에서 **자동 차감** | TADA 정산 → 임베디드 파이낸스 차감 → 상환 처리 | 프놈펜 [[TADA]] 기사가 전기오토바이 할부를 라이드 수익에서 차감 |
| `WALLET` | [[지급인 Payer]]가 본인 크립토 지갑으로 **온체인 직접** 송금 | [[Payback Escrow]] 컨트랙트로 USDC 전송 | USDC 결제에 익숙한 채무자가 직접 입금 |
| `GUARANTOR` | 처음부터 [[보증인 Guarantor]]가 **대납하기로** 약속된 케이스 | Guarantor 자금이 [[Payback Escrow]]로 흘러옴 | 채무자가 갚을 의사가 약해 사실상 보증인이 진행 |

> [!note] 가장 흥미로운 건 `TADA`
> `TADA`는 단순 결제 수단이 아니라 **MVL이 운영하는 동남아 차량공유 슈퍼앱**이다. 캄보디아 [[TADA]] 기사들에게 전기오토바이/렌트비 등을 할부로 팔고, 매일/매주 라이드 수익에서 자동 차감하는 임베디드 파이낸스(embedded finance) 구조. 이게 [[ar-service]]가 단순 RWA 플랫폼이 아니라 **MVL 본업 생태계와 직결된 금융 레이어**라는 결정적 증거다.

#### 왜 메타데이터로만 박나?

```
  [Invoice 생성]
        │
        ▼
  defaultPaymentMethod = "TADA"   ← DB(Payer 등록 시 입력된 값)
        │
        ▼
  invoice-ipfs.service.ts:154
        │
        ▼
  {
    "attributes": [
      { "trait_type": "defaultPaymentMethod", "value": "TADA" },
      ...
    ]
  }
        │
        ▼
   IPFS 업로드 → ipfsCid 발급
        │
        ▼
   온체인 NFT 메타데이터로 영구 기록
```

> [!tip] 왜 굳이 NFT 속성으로?
> 채권이 [[조각 fraction]] 단위 NFT로 거래되는데, **각 NFT가 "이 채권은 TADA 자동 차감 기반"이라는 정보를 들고 다녀야** 투자자가 2차 시장(향후 가능 시)에서도 위험을 판단할 수 있다. 시스템 분기는 사람(운영자)이 하고, NFT는 그 사실을 들고 다니며 신뢰를 운반하는 라벨 역할.

---

### 2. MVL의 수익 모델 — 코드로 확인 vs 비즈니스 추론

이제 본 주제. **MVL은 [[ar-service]]를 가지고 어떻게 돈을 버나?**

두 트랙으로 나눠야 정직하다.

#### 트랙 A — 코드로 확인되는 4가지

| # | 이름 | 어디서 떼나 | 위치/근거 |
|---|---|---|---|
| ① | **플랫폼 수수료 (Platform Fee)** | 투자자 **이익**의 일정 % | `PLATFORM_FEE_PERCENT` 환경 변수, `purchase-fee.service.ts` |
| ② | **프로토콜 수수료 (Protocol Fee)** | 투자자 redeem 시 온체인 컨트랙트가 차감 | `FractionsRedeemed` 이벤트의 `protocolFeeUsdc` 필드 |
| ③ | **MVL 보증인 fee** | MVL이 직접 [[보증인 Guarantor]]일 때 보증료 | `isInternalMvl` 플래그, `GuaranteePosition.fee` |
| ④ | **MVL 판매자 오리지네이션** | MVL이 직접 [[판매자 Seller]]일 때 채권을 만들어서 팔아 차익 | `isMvlInternal` 플래그 |

##### ① 플랫폼 수수료

```
guaranteeFee   = totalProfit × (guaranteeFee% / 100)   ← 보증인 몫
platformFee    = totalProfit × (PLATFORM_FEE_PERCENT)  ← MVL 몫
net to investor = faceValue − guaranteeFee − platformFee
```

> [!warning] 현재 `PLATFORM_FEE_PERCENT` 기본값은 **0%**
> 코드상 떼는 메커니즘은 박혀 있지만 **현재 기본값은 0%**다. 운영자가 환경 변수로 언제든 켤 수 있는 "다이얼"이지 지금 당장 주 수익원은 아니다. 이게 MVL이 **수수료로 돈 버는 회사가 아니라는 결정적 단서**.

##### ② 프로토콜 수수료 (`protocolFeeUsdc`)

투자자가 만기 후 [[redeem]]할 때 온체인 컨트랙트(`MusubiARProtocol`)가 `FractionsRedeemed` 이벤트를 발행하는데, 이벤트에 `redemptionAmountUsdc / protocolFeeUsdc / guarantorFeeUsdc` 3개가 함께 디코드된다. 즉 **온체인 레이어에서 한 번 더 떼는 수수료**가 존재한다. [[ar-service]]는 그 값을 받아 기록만 한다.

##### ③ MVL 보증인 fee (`isInternalMvl`)

MVL이 [[Guarantor Registry]]에 직접 자기 자신을 보증인으로 등록한 케이스. 이 경우 보증료(`GuaranteePosition.fee`) — **투자자 이익의 N%** — 가 MVL로 들어온다.

```
예: faceValue 300만, 매입가 280만 → 이익 20만
    fee 5% → guaranteeFee = 1만원 → MVL 수익
```

> [!note] 시나리오 A에서 보면
> [[TADA]] 기사 소페아의 채권에 MVL 자신이 보증인으로 들어가면, MVL은 (a) [[TADA]] 본업으로 기사를 잘 알아서 정확한 위험 평가가 되고, (b) 보증료로 돈을 벌고, (c) 못 갚으면 [[TADA]] 수익 차감으로 회수까지 한다. **위험 통제와 수익이 같은 흐름에 묶이는 게 핵심.**

##### ④ MVL 판매자 오리지네이션 (`isMvlInternal`)

MVL이 [[판매자 Seller]]로 등록되어 채권 자체를 만들어 파는 경우. 채권 액면가와 [[Purchase Escrow]]에서 받는 금액의 차이만큼 자금 조달 비용 차익 — 즉 **오리지네이션 수익**.

#### 트랙 B — 코드엔 직접 안 박혔지만 비즈니스 모델상 추론되는 것

> [!example] 코드만 보면 "MVL은 수수료가 0%인데 뭘로 돈 벌지?"
> 답은 **이 플랫폼 자체가 본업이 아니라 본업을 키우는 도구**라는 것. 진짜 돈은 본업에서 난다.

| # | 이름 | 어떻게 버나 |
|---|---|---|
| ⑤ | **예대 스프레드** | MVL이 동시에 판매자/보증인/오리지네이터로 끼면 채권을 싸게 만들어 비싸게 파는 마진 확보 |
| ⑥ | **TADA 본업 시너지 + lock-in** | [[TADA]] 기사에게 차량/오토바이 할부 → 기사가 [[TADA]]에 묶임 → 라이드 매출 ↑ + 이탈률 ↓ |
| ⑦ | **MVL 토큰 효용** | [[Guarantor Vault]]가 USDC/MVL을 담보로 받음 → MVL 토큰 수요/효용 ↑ |

##### ⑥ 본업 시너지가 가장 큰 그림

```
                    ┌─────────────────────────────────────┐
                    │              MVL 그룹                 │
                    │                                      │
   기사 소페아 ────► │ [TADA 앱]  ────────────────┐         │
   (Payer)          │     │                       │         │
                    │     │ 라이드 수익 차감       │         │
                    │     ▼                       │         │
                    │ [ar-service] ◄── 보증인 fee ─┤         │
                    │     │                       │         │
                    │     │ 채권 토큰화           ▼         │
                    │     ▼                    [메콩모터스]   │
                    │  [투자자에게 판매] ─► 현금  │         │
                    └──────────┬──────────────────┘
                               ▼
                       투자자 (외부)
```

이 그림에서 MVL이 돈 버는 곳은 **수수료 박스가 아니다**. 진짜 보너스는:

1. 소페아가 전기오토바이를 살 수 있게 됨 → [[TADA]] 기사 풀 확장
2. 소페아의 라이드 수익에서 자동 차감 → [[TADA]] 매출에 묶임 → 이탈 못함
3. 채권은 외부 투자자가 자금을 댐 → MVL은 자본 부담 없이 기사 풀 키움
4. 그러는 사이 보증료·플랫폼 수수료는 옵션으로 켤 수 있는 다이얼로 남겨둠

##### ⑦ MVL 토큰 효용

[[Guarantor Vault]]는 USDC와 함께 **MVL 토큰**도 담보로 받는다 (`SupportedTokenConfig: BASE_MVL/MVL → GuarantorVaultToken.MVL`). 보증인이 늘수록 → MVL 토큰이 vault에 잠겨 락업 → 토큰 효용·수요·가격 안정성에 기여.

> [!warning] 단, MVL 토큰 가치는 [[ar-service]]가 평가하지 않음
> [[ar-service]]는 MVL→USD 환산을 **직접 하지 않는다**. `vaultContract.read.totalUSDValue(bytes32 guarantorId)`로 **온체인 컨트랙트가 USDC 6 decimals 기준으로 환산**한 값을 받아오기만 한다. 가격 변동성이 크기 때문에 항상 온체인 실시간 평가가 안전.

---

### 3. 본질 — "금융 끼워넣어 본업 생태계 키우기"

> [!tip] 한 줄 요약
> MVL의 수익 모델은 **수수료 비즈니스가 아니라 본업 부스터**다. [[ar-service]]는 외부 자본을 끌어와 [[TADA]] 기사들에게 자산을 깔아주는 **임베디드 파이낸스 레이어**이고, MVL은 그 과정에서 옵션적인 4가지 수수료를 떼면서 본업의 lock-in과 토큰 효용을 동시에 키운다.

코드만 보면 "수수료 0%인데 어떻게 회사가 돌아가?"라는 의문이 들지만, **`TADA`라는 단어 하나가 모든 걸 설명**한다. `defaultPaymentMethod = TADA`가 IPFS NFT 속성으로 박히는 이유, MVL이 보증인으로 직접 들어가는 이유, 플랫폼 수수료를 0%로 둬도 괜찮은 이유 — 전부 본업이 진짜 엔진이기 때문.

## 비유로 보는 전체 플로우 (아기 버전 + brain rot)

### 아기 버전 — 동네 문방구 이야기

옛날 옛적, 작은 마을에 문방구가 있었어요. 이 마을에서 일어나는 일을 따라가면 [[ar-service]]가 무엇을 하는지 한 번에 알 수 있어요.

> [!example] 등장 친구들 정리
> - 🍪 **과자가게 사장님** = [[seller]] (외상으로 물건 팔고, 돈 빨리 받고 싶음)
> - 🧒 **외상 친구** = [[payer]] (다달이 용돈으로 갚는 채무자)
> - 📜 **외상 쪽지** = [[invoice]] ("한 달 뒤 1000원 줄게" 적힌 종이)
> - 💸 **투자 친구** = [[investor]] (쪽지를 싸게 사서 만기에 비싸게 받기)
> - 💪 **든든한 형아** = [[guarantor]] (외상 친구가 못 갚으면 대신 갚아주기로 약속)
> - 🐷 **형아 돼지저금통** = [[guarantor-vault]] (형아가 미리 동전 넣어둔 금고)
> - 🐰 **공책 비서 토끼** = [[ar-service]] (돈은 안 만지고 공책만 정리)
> - ✨ **마법 금고** = [[ar-protocol-contract]] (자판기처럼 규칙대로만 돈이 나가는 곳)

#### 1️⃣ 외상 쪽지가 태어나요 — 그리고 케이크처럼 10조각으로 잘라요

외상 친구가 사장님한테 1000원짜리 과자를 외상으로 받았어요. 사장님은 "한 달 뒤 1000원 줄게" 적힌 [[invoice|쪽지]]를 받죠. 근데 사장님은 한 달이나 못 기다려요. 그래서 이 쪽지를 **케이크 자르듯 10조각**으로 자릅니다. ([[fraction]])

```
   [ 외상 쪽지 1000원 ]
          ↓ 케이크 자르기
   🍰🍰🍰🍰🍰🍰🍰🍰🍰🍰
   (한 조각 = 100원짜리)
```

> [!tip] 왜 자를까?
> - **용돈만 있어도 살 수 있어요** — 1000원 통째로 못 사도 100원짜리 한 조각은 살 수 있죠.
> - **위험을 나눠요** — 한 사람이 다 사면 망했을 때 혼자 울지만, 10명이 나눠 사면 살짝씩만 슬퍼요.
> - **조각마다 도장(token)** — 누가 어느 조각을 샀는지 마법 금고가 다 기억해요.

#### 2️⃣ 신호등 단계 — 쪽지가 자라나는 다섯 표정

쪽지는 태어나자마자 진열대에 못 올라가요. 단계별로 옷을 갈아입어야 해요. 이게 **신호등** 같은 [[invoice-status|단계(status)]]예요.

```
🛌 DRAFT (낮잠)
   ↓ "확인했어요!"
🧼 CONFIRMED (세수하고 옷 입기)
   ↓ "사진 찍어 사물함에 넣고, 마법 공책에 도장 받기"
🟢 LISTED (진열대 등장!)
   ↓ "투자 친구 한두 명이 사감"
🟡 PARTIALLY_FUNDED (조각 절반 팔림)
   ↓ "전부 팔림!"
🔴 SOLD_OUT (다 팔렸어요)
```

> [!note] 신호등은 왜 필요할까?
> 빨간불일 땐 살 수 없고, 초록불(`LISTED`)일 때만 살 수 있어요. 단계가 없으면 아직 도장 못 받은 가짜 쪽지도 팔릴 수 있겠죠? 신호등이 **안전 규칙**을 만들어줘요. 갚는 단계(`UNPAID → PARTIALLY_PAID → OVERDUE → MISSED → PAID/DEFAULTED`)도 똑같은 신호등이에요.

#### 3️⃣ 인터넷 사물함 번호표 — IPFS

쪽지 원본을 마법 공책(블록체인)에 통째로 넣으면 무거워서 못 들어요. 그래서 쪽지 **사진**을 인터넷 사물함([[IPFS]], [[Pinata]])에 넣고 **번호표** ([[ipfsCid]])를 받아와요.

```
   📜 쪽지 원본
       ↓ 사진 찰칵
   ☁️  인터넷 사물함(IPFS)
       ↓ 보관증 발급
   🎫 번호표: "Qm...abc123"
       ↓
   ✨ 마법 공책에는 번호표만 적기
```

> [!warning] 번호표 없으면 등록 못해요
> 번호표가 없으면 마법 금고가 "가짜 쪽지일 수도 있어!" 하고 거절해요. 그래서 쪽지가 LISTED로 가기 **전에 반드시** 번호표를 받아와야 합니다.

#### 4️⃣ 형아 돼지저금통 — 담보 금고

[[guarantor|형아]]가 "외상 친구 못 갚으면 내가 80%까지 대신 갚을게!" 약속하면, 그 약속을 지킬 수 있다는 증거로 **돼지저금통**에 미리 동전을 넣어둬요.

```
   🐷 형아 돼지저금통 ([[guarantor-vault]])
   ┌──────────────────────────┐
   │  💵 USDC 동전 (안정적)    │
   │  🪙 MVL 동전 (값 변동)    │
   └──────────────────────────┘
        ↑          ↑
        │          └── 매일 검사 선생님
        │              (CRON_AR_SERVICE_CHECK_GUARANTOR_COLLATERAL)
        │              "7일 안에 갚아야 할 쪽지들 합쳐서
        │               저금통에 그만큼 있나요?" 검사
        │
        └── 도둑 감시 카메라
            (MonitorVaultMovementUc)
            "이상한 지갑이 가져가면 슬랙 알림!"
```

> [!tip] MVL 동전 값은 누가 알아요?
> 비서 토끼는 MVL 값을 직접 못 매겨요. 마법 금고에 "지금 이 저금통 USD로 얼마예요?" ([[totalUSDValue]]) 직접 물어보면 답해줍니다. 가격이 흔들거리니까 실시간 답이 가장 안전해요.

#### 5️⃣ 가격 전략과 편지함 — 어떻게 팔지 정하기

쪽지를 어떻게 깎아 팔지 사장님이 미리 골라둬요. 세 가지 **녹는 아이스크림 전략**이 있어요.

| 전략 | 비유 | 동작 |
|------|------|------|
| `FIXED_APR` | 정찰제 🏷️ | 언제 사도 똑같은 % |
| `LINEAR_DECAY` | 미끄럼틀 🛝 | 시간 지날수록 일직선으로 이자 줄어듦 |
| `STEP_DECAY` | 계단 🪜 | 매주/매달마다 뚝뚝 떨어짐 |

그리고 마법 공책에 도장 찍으러 갔는데 가끔 길이 막혀요. 그럴 때 **편지함**([[outbox]], [[OnchainTransaction]])에 할 일을 적어두면 비서 토끼가 **5번까지** 다시 시도해줘요.

```
   📬 편지함
   ┌─────────────────────────────┐
   │ [PENDING] 마법 공책에 도장   │
   │ [PROCESSING] 찜해놓고 시도중 │
   │ [COMPLETED] 성공!            │
   │ [FAILED → 다시 PENDING]      │
   │ [PERMANENTLY_FAILED 5번 실패]│
   │  └─ 슬랙으로 어른 호출 📢    │
   └─────────────────────────────┘
```

#### 6️⃣ 매일 검사 — 형아 약속 지킬 수 있나?

매일 검사 선생님(크론)이 일어나서 묻습니다:

> "오늘부터 7일 안에 만기 오는 쪽지들 다 합치면 얼마인가요? 그 80%만큼 형아 저금통에 있나요?"

부족하면 형아한테 이메일 슝~ "동전 더 넣어주세요!"

#### 7️⃣ 해피엔딩과 슬픈 결말

```
   ┌──── 외상 친구가 갚으면 (해피 😊) ────┐
   │                                       │
   외상 친구 → 💵 Payback Escrow (상환 금고)
              ↓ 운영자가 payment event 기록
              ↓ 비서 토끼가 마법 공책에 sync
              ↓ 만기 되면 redeem 창구 열림
   투자 친구 ← 💵 (이자 포함해서 회수)
   
   
   ┌──── 외상 친구가 못 갚으면 (슬픔 😢) ─┐
   │                                       │
   외상 친구 → ❌ (돈 안 보냄)
              ↓ DEFAULTED 판정
              ↓ 마법 금고가 자동 발동
   🐷 형아 돼지저금통 깨기!
              ↓ coverageRatio(80%)만큼
   투자 친구 ← 💵 (대부분 회수, 약간 손실)
```

> [!note] 비서 토끼는 돈을 만지지 않아요
> 진짜 돈은 **마법 금고 3형제**가 움직여요:
> - [[purchase-escrow]] = 투자 친구가 낸 돈을 모아 사장님에게 전달
> - [[payback-escrow]] = 외상 친구가 갚은 돈을 모아 투자 친구에게 전달
> - [[guarantor-vault]] = 형아 담보 보관
>
> 비서 토끼([[ar-service]])는 공책만 정리하고, 도장만 찍어달라고 부탁할 뿐이에요.

#### 8️⃣ 한 장 요약 다이어그램

```
                  🐰 비서 토끼 (ar-service)
                  "공책 정리만 해요"
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
   📜 쪽지 만들고   🎫 번호표 받고   📬 편지함 통해
   (Contract→        (IPFS)           도장 부탁
    Invoice)                          (Outbox→Onchain)
                       │
                       ▼
              ✨ 마법 금고 (Smart Contract)
              "자판기처럼 규칙대로만 움직임"
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
   💵 Purchase     💵 Payback      🐷 Guarantor
      Escrow          Escrow           Vault
   (투자금)        (상환금)         (담보)
```

---

### Brain Rot 버전 (옵션 챕터, 알람 끄고 보세요)

> [!warning] 진지함 0%
> 이 단락은 그냥 재미용. 위의 아기 버전이 진실의 모든 것.

옛날 옛적에 과자가게 ㅇㅇ 사장님(개불쌍 [[seller]], 현금 부족 어휴), 외상충 [[payer]](용돈 받으면 갚는다고 함, 진짜?), 외상 쪽지가 ✨NFT✨로 환생해서 [[invoice]]가 됨. 케이크처럼 10조각 ㄱㄱ ([[fraction]]) — 한 조각씩 사 모으는 투자 brO들([[investor]]) 등장. 950원에 사서 1000원 받으면 +50원 ez 개꿀 [[APR]] 떡상 노캡 fr.

근데 외상충이 튈 수도 있잖아? 그럼 두 가지 길:
- **노가드 비보증** ([[isGuaranteed|isGuaranteed=false]]): 손해 그대로 떠안음 L bozo
- **시그마 형아 보증** ([[guarantor]]): 형아가 돼지저금통([[guarantor-vault]])에 USDC/MVL 미리 넣어두고 "맡겨" 함. fanum tax 받는 대신 외상충 튀면 저금통 깨서 보상 ㅇㅇ

저금통 매일 검사 ㄱㄱ (`CRON_AR_SERVICE_CHECK_GUARANTOR_COLLATERAL`) — 7일 안에 갚을 거 합쳐서 80% 안 차있으면 이메일 슝슝 "동전 더 넣으셈". MVL 값은 마법 금고가 직접 알려줌 ([[totalUSDValue]]) 왜냐면 코인값 위아래 미친듯이 흔들림 ㅇㅇ.

도둑도 감시함 ([[MonitorVaultMovementUc]]) — 모르는 지갑이 저금통 건드리면 among us SUS 🚨 슬랙 `@channel` ㄷㄷ. 이상한 거 발견하면 그 자리에서 임포스터 추방.

가격 전략 3종 ㄱㄱ:
- `FIXED_APR` 정찰제 (no cap, no rizz)
- `LINEAR_DECAY` 미끄럼틀 (시간 갈수록 떡락)
- `STEP_DECAY` 계단 (주마다 뚝)

마법 공책에 도장 못 찍으면? [[outbox|편지함]]에 적어두고 5번까지 재시도 — 5번 다 실패하면 `PERMANENTLY_FAILED` 떠서 슬랙 알림 ㄱㄱ. 돈 관련 작업이 조용히 사라지면 그건 NPC 무브임 안 됨.

외상충이 갚으면? Payback Escrow에 돈 차곡차곡 → 투자 brO들 만기에 redeem 창구 열림 → 💰 회수. **튀면?** 마법 금고가 형아 저금통 부숨 (fanum tax 진심 ver) → coverageRatio(80%) 만큼 보상. 비보증이면 그냥 L take.

핵심 ㄱㄱ:
- [[ar-service]] = 공책 비서 (돈 안 만짐, just vibes 정리)
- [[ar-protocol-contract|마법 금고]] = 진짜 마법사 (코드가 곧 법, 자판기 무브)
- [[TADA]] = MVL 본업 (캄보디아 차량공유 슈퍼앱, 기사 수익에서 자동 차감 = `defaultPaymentMethod=TADA`)

전체 플로우 한 줄: **사장님 현금 급함 → 외상 쪽지 NFT화 → 조각냄 → 형아가 담보 빵빵하게 → 투자 brO들 조각 줍줍 → 만기에 회수 ✨ 혹은 디폴트 시 형아통 깨서 보상**. 이게 RWA 인보이스 투자 상품 노캡 fr fr 🗿.

---

## English

### TL;DR

[[AR-Service]] is a platform that **tokenizes and fractionalizes [[RWA]] invoices (accounts receivable)** into investment products. There are **5 actors** — the [[Operator]] runs whitelist registration, the [[Seller]] sells receivables to get cash immediately, the [[Payer]] pays down the debt monthly until maturity, the [[Investor]] buys fractions to capture the [[APR]] spread, and the [[Guarantor]] absorbs default risk in exchange for a fee. Actual funds live in the **three on-chain vaults** — [[Guarantor Vault]] (collateral), [[Purchase Escrow]] (investor→seller purchase funds), and [[Payback Escrow]] (payer→investor repayments) — while [[AR-Service]] itself **never touches money**: it acts as the **recorder and coordinator**, handling IPFS upload, on-chain registration, outbox retries, and settlement event decoding. The **guaranteed/unguaranteed axis** drives the risk-return profile: [[isGuaranteed=false]] means the investor eats all losses but earns a higher [[APR]], while [[isGuaranteed=true]] means [[Guarantor Vault]] collateral covers losses up to [[coverageRatio]] in exchange for a lower [[APR]] minus a [[guaranteeFee]] skimmed from profit.

> [!note] One-liner
> **[[AR-Service]] = the "notebook secretary" for a market that slices receivables into NFT fractions; real money sits in the [[on-chain vault trio]]; with a [[Guarantor]] → safe & low-yield, without → risky & high-yield.**

> [!example] 30-second picture
> ```
>                      ┌──────────────────────────────┐
>                      │   AR-Service (recorder)      │
>                      │  IPFS · Outbox · Aggregator  │
>                      └───────┬───────────────┬──────┘
>                              │ register      │ sync
>                              ▼               ▼
>   [Seller] ── receivable ──▶ [Invoice × 10] ──▶ [Investor]
>      ▲                         │                 │
>      │ instant cash             │ default         │ redeem at maturity
>      │                         ▼                 ▼
>   [Purchase Escrow] ◀── [Payback Escrow] ◀── [Payer] monthly repayment
>                                    ▲
>                                    │ shortfall cover (coverageRatio %)
>                                    │
>                            [Guarantor Vault] ◀── [Guarantor] (collateral)
> ```

| Axis | [[Unguaranteed]] (`isGuaranteed=false`) | [[Guaranteed]] (`isGuaranteed=true`) |
|---|---|---|
| Risk bearer | Investor (full) | [[Guarantor]] covers up to [[coverageRatio]] |
| [[APR]] | High | Low |
| Listing path | `LISTED` right after SC register | `LISTED` only after `GuaranteePosition` on-chain register |
| Extra fee | None | [[guaranteeFee]] = investor profit × fee% |
| On default | Loss as-is | Contract pulls collateral from [[Guarantor Vault]] |

### Core Terminology (APR · Invoice · Fraction · IPFS Tag)

Four words show up over and over in this note. Pin them down now, or [[가격-전략]], [[보증-구조]], and the [[redeem-창구]] discussion later will turn into mush.

---

### 1) APR (Annual Percentage Rate, simple-interest annual yield)

> [!note] One-liner
> **APR = annualized simple-interest rate.** It strips out compounding (interest earning interest) and gives you the raw 1-year yield (or borrowing cost).

The simplest example:

```
Deposit 1,000,000 KRW for 1 year → get 50,000 KRW back
APR = 50,000 / 1,000,000 = 5%
```

In [[ar-service]], an [[Invoice]] stores `apr` directly or expresses it via `minApr`/`maxApr` inside a [[가격-전략]]. Human-facing values are percent (e.g., `7.5`); on-chain values are multiplied by 100 to become **bps (basis points, 750)** — this dodges float rounding errors (`invoice-onchain.service.ts:127`).

#### APR vs APY — don't confuse them

| Item | APR | APY |
|---|---|---|
| Interest calc | **Simple** | **Compound** |
| Compounding effect | Ignored | Included |
| Headline value at same rate | Lower | Higher |
| Used in AR-Service | ✅ APR | ❌ |

> [!tip] Why simple-interest APR is enough for AR invoices
> An invoice typically matures in **30–90 days**, rarely beyond a year. At those horizons the difference between simple and compound interest is negligible, and more importantly the structure is **lump-sum at maturity** — interest never gets reinvested mid-flight, so there is no compounding to speak of.

#### APR breathes with time-to-maturity

The same face-value [[Invoice]] yields a different APR depending on **when** you buy it. The closer to maturity, the less risk you carry, so APR naturally drops — this is exactly what [[LINEAR_DECAY]] and [[STEP_DECAY]] strategies (covered in the [[가격-전략]] section) encode.

---

### 2) Invoice — a tokenized account receivable (not just a "bill")

> [!warning] "Invoice = bill" is only half right
> In the [[ar-service]] context, an **Invoice is a tokenized Account Receivable (AR)** that gets traded. It is not a sheet of paper — it is **a financial instrument registered on-chain and sliced into fractions for sale**.

#### The essence — selling "the right to receive later" today

```
[Seller] ── sells goods/service ──> [Payer]
   │                                    │
   └── right to 1,000 in 60 days ◀──────┘
         (this is the AR = Invoice)
                 │
                 ▼
   "I'll take 950 now and hand you the right at maturity"
                 │
                 ▼
          [Investor] buys at 950
                 │
        60 days later collects 1,000 → +50 profit = APR
```

The [[Seller]] gets **instant cash** instead of waiting for maturity; the [[Investor]] earns the **discount spread** between purchase price and face value. [[ar-service]] records and orchestrates the dance in between.

#### Key fields (per DB schema)

| Field | Type | Meaning |
|---|---|---|
| `invoiceId` | Int (starts at 100000) | System identifier |
| `faceValue` | Decimal(20,6) | **Amount due at maturity** |
| `dueDate` | DateTime | Maturity date |
| `fractionCount` / `availableCount` | Int | Total fractions / remaining |
| `isGuaranteed` | Boolean | Whether [[보증-구조]] applies |
| `strategy` / `strategyParams` | Enum / JSON | [[가격-전략]] (FIXED_APR / LINEAR_DECAY / STEP_DECAY) |
| `ipfsCid` | String | [[IPFS]] metadata tag — required before on-chain registration |
| `txHash` | String | On-chain registration tx hash |
| `tokenIds` | Int[] | NFT token IDs minted per fraction |

#### State machine — two parallel traffic lights

An Invoice moves along **two independent axes**: "sales progress" and "repayment progress".

```
[Sales axis] InvoiceSavedStatus
  DRAFT ──► CONFIRMED ──► LISTED ──► PARTIALLY_FUNDED ──► SOLD_OUT
  (born)    (IPFS done)   (on sale)   (some sold)         (all sold)

[Repayment axis] InvoicePaymentStatus
  UNPAID ──► PARTIALLY_PAID ──► PAID         (happy ending)
       └──► OVERDUE ──► MISSED ──► DEFAULTED  (default)
```

> [!example] Why two axes
> "Did investors buy it all (sales)?" and "Did the payer pay it back (repayment)?" are completely independent events. `SOLD_OUT && PARTIALLY_PAID` is natural; so is `LISTED && UNPAID`.

---

### 3) Fraction — why slice it like a cake

A single [[Invoice]] is never sold whole — it is cut into **`fractionCount`** pieces. A 1,000-unit face-value note sliced into 10 fractions = 100 per fraction.

```
            Invoice (faceValue 1,000)
                       │
       ┌───────┬───────┼───────┬───────┐
       ▼       ▼       ▼       ▼       ▼
     frac1   frac2   frac3   ...   frac10
     (100)   (100)   (100)         (100)
       │       │       │             │
       ▼       ▼       ▼             ▼
   InvestorA InvestorA InvestorB    InvestorC
   (one investor can hold many fractions — 1:N relationship)
```

#### Why split at all? Three reasons

> [!tip] Three benefits of fractionalization
> 1. **Small-ticket entry** — investors can join a 1,000-unit note with as little as 100 → lower barrier
> 2. **Risk diversification** — investors don't go all-in on a single note; they spread across many notes and fractions
> 3. **Digital stamps (NFTs)** — each fraction is an on-chain token (`tokenIds Int[]`). Who owns which fraction is etched into the magic notebook (blockchain), making forgery and double-selling impossible

#### Per-fraction pricing formula

```
Fraction price = faceValue ÷ fractionCount
              (then discounted at the strategy's APR for the current moment)

e.g.) faceValue 1,000, fractionCount 10
      Per-fraction face = 100
      Purchase price after APR ≈ 95 (assuming 30 days to maturity, ~60% APR)
```

Sales progress is tracked via `availableCount`. All fractions sold → `SOLD_OUT`; some sold → `PARTIALLY_FUNDED`. The auto-transition `LISTED → PARTIALLY_FUNDED → SOLD_OUT` fires per fraction sold.

---

### 4) ipfsCid — the internet-locker claim ticket

> [!note] One-liner
> **ipfsCid is the receipt ID for the Invoice metadata uploaded to IPFS (InterPlanetary File System).** Instead of storing heavy metadata on the magic notebook ([[블록체인]]), we store only **this claim ticket**.

#### Why IPFS, why only the tag on-chain

```
[Invoice metadata: faceValue, dueDate, paymentMethod, ...]
                       │
                       ▼
            Uploaded to [Pinata IPFS]
                       │
                       ▼
        Returns ipfsCid: "QmXk7Ya..."  ← claim ticket
                       │
                       ▼
       Only this tag is written on-chain
       (the actual metadata stays on IPFS)
```

> [!tip] Why not store the body on-chain
> 1. **Gas cost** — putting a large JSON on-chain is expensive. A 32-byte tag is cheap.
> 2. **Immutability proof** — IPFS CIDs ARE hashes of the content. Same CID = mathematically guaranteed same content, bit for bit.
> 3. **Tamper detection** — alter the file on IPFS and the CID changes, breaking the on-chain match. Forgery becomes self-evident.

#### "No tag, no on-chain registration" — a hard gate

In [[ar-service]]'s pipeline, `ipfsCid` is an **explicit prerequisite**:

```
generateInvoices
   → confirmAllInvoices (DRAFT → CONFIRMED)
   → IPFS upload → obtain ipfsCid   ◀── cannot proceed without this
   → SC Register (on-chain registration)
   → (if guaranteed) GuaranteePosition registration
   → LISTED (appears on the market)
```

> [!warning] The real reason ipfsCid is mandatory
> The on-chain registration (`SC_REGISTER`) transaction takes `ipfsCid` as a literal input parameter. Without the tag, the contract call simply cannot be constructed. In other words, **"proof that the real receivable metadata is permanently archived somewhere"** is a precondition for an [[Invoice]] to hit the market. It's the first line of defense against fake notes being listed.

#### What lives inside the metadata

A near-NFT-standard property bundle (`invoice-ipfs.service.ts:154`):

- `faceValue`, `dueDate`, `fractionCount`, etc. — basic receivable info
- `defaultPaymentMethod` (FIAT_SELLER / TADA / WALLET / GUARANTOR — see the [[수익-모델]] section)
- `isGuaranteed`, `coverageRatio`, `guaranteeFee` (when guaranteed)
- Seller / Payer / Guarantor `externalRef` and other identifiers

This bundle lives on IPFS forever — anyone with the CID can verify the receivable's original spec.

---

### One-page recap

```
┌──────────────────────────────────────────────────────────────┐
│  APR        = simple-interest annual yield (NOT APY)         │
│  Invoice    = tokenized AR (DRAFT→…→SOLD_OUT, UNPAID→…)      │
│  Fraction   = slice = NFT stamp (small-ticket·diversify·anti-fraud) │
│  ipfsCid    = IPFS tag (mandatory gate for on-chain register) │
└──────────────────────────────────────────────────────────────┘
```

These four words are the skeleton. From the next section on, we stack [[등장-인물-5명]], [[가격-전략]], [[보증-구조]], and [[금고-3형제]] on top of it.

### The Five Actors with Real-World Examples

The first thing that trips people up about [[AR-Service]]–based [[RWA]] invoice products is **who is who**. Conflating "Seller" with "Payer" in particular collapses the whole flow. This section nails down the five actors and walks through two concrete scenarios so it's unambiguous.

### Five Actors at a Glance

| # | Role | Korean | One-line definition | Where they live in the system |
|---|------|--------|---------------------|--------------------------------|
| 1 | **Operator / Admin** | 운영자 | The [[MVL]]-side party that vets and registers [[Seller]]/[[Payer]]/[[Guarantor]] and coordinates settlement | `admin-service`, `@TrustedController` calls |
| 2 | **[[Seller]]** | 판매자 | The creditor who holds the "right to be paid" and wants to **cash out early** by selling the receivable | Whitelisted in [[SellerRegistry]] |
| 3 | **[[Payer]]** | 지급인 | The debtor who **owes money and pays monthly** (or at maturity) | Whitelisted in [[PayerRegistry]], carries `defaultPaymentMethod` |
| 4 | **[[Investor]]** | 투자자 | Buys [[Fraction]]s at a discount up front, receives face value at maturity → earns the [[APR]] spread | No on-chain registry, just profile + JWT |
| 5 | **[[Guarantor]]** | 보증인 | An insurer-like party that covers investor losses up to `coverageRatio` via [[Guarantor Vault]] collateral if the Payer defaults | Whitelisted in [[GuarantorRegistry]] + on-chain collateral |

> [!warning] The biggest pitfall: Seller ≠ Payer
> If you read "the Seller pays monthly," **the whole flow flips upside down**.
> - **Seller** = creditor who **sells the receivable to get cash now** (money flows IN)
> - **Payer** = debtor who **owes the money and pays it back** (money flows OUT, monthly)
>
> These are **different people**. One invoice = one month's installment, and the one who pays it back is **always the Payer**.

---

### Scenario A — A TADA Driver's Electric Scooter Installment

Given that [[MVL]]'s core business is the Southeast-Asian ride-hailing super-app [[TADA]], this is the most natural real-world example.

```
┌──────────────────┐         ┌──────────────────┐
│  Mekong Motors   │  sells  │   Sopheap         │
│  (EV scooter     │ ──────▶│  (Phnom Penh      │
│   dealer)        │  3M KRW │   TADA driver)   │
│   = Seller       │  12-mo  │   = Payer         │
│                  │  install│                  │
└────────┬─────────┘         └────────▲─────────┘
         │                              │
         │ Receivable                  │ 250k KRW / month
         │ → 12 invoices               │ (auto-deducted from
         │                              │  TADA earnings)
         ▼                              │
┌──────────────────┐         ┌──────────────────┐
│  Jihoon          │  buys   │  Angkor Capital   │
│  (Korean USDC    │ fraction│  (Cambodian       │
│   investor)      │  for    │   credit guarantor)│
│   = Investor     │  2.8M   │   = Guarantor     │
└──────────────────┘          └──────────────────┘
```

**What each actor does:**

| Actor | Real-world action | System action |
|-------|-------------------|----------------|
| Mekong Motors (Seller) | Sells scooter to Sopheap on 3M KRW credit, urgently needs 2.8M cash now | Registered in [[SellerRegistry]] → receives early payout from [[Purchase Escrow]] |
| Sopheap (Payer) | Works as a TADA driver, pays 250k KRW/month for 12 months. `defaultPaymentMethod=TADA` | Registered in [[PayerRegistry]], `externalRef` links to TADA driver ID |
| Jihoon (Investor) | Buys part of the 10 fractions for 2.8M KRW → recovers 3M KRW at maturity (~200k profit) | Deposits USDC to [[Purchase Escrow]], redeems after maturity |
| Angkor Capital (Guarantor) | Covers up to 2.4M KRW (=3M × 80%) if Sopheap defaults, earns a fee in return | Posts collateral in [[Guarantor Vault]], `coverageRatio=80`, `fee=5` |
| MVL (Operator) | KYC's and vets all three, coordinates fractionalization and on-chain registration | Registers actors in registries via [[Admin Service]], creates [[Contract]] |

> [!example] Money direction in one line
> **Mekong Motors → scooter → Sopheap**, **Sopheap → 250k/month → Payback Escrow → Jihoon**, **Jihoon → 2.8M → Purchase Escrow → Mekong Motors (early settlement)**.
> So **the Seller gets paid up front**, **the Payer pays monthly**, **the Investor pays up front and recovers at maturity**.

---

### Scenario B — Coffee Farm and Café Chain

A more B2B / trade-finance-flavored example. It mirrors the classic SME pain point in emerging markets: **profitable on paper but bankrupt in cash**.

```
┌──────────────────┐         ┌──────────────────┐
│  Coffee Farm     │  sells  │   Seoul Café      │
│  (Vietnam SME)   │ ──────▶│   Chain (50       │
│   = Seller       │  50M KRW│   locations)     │
│                  │  net 90 │   = Payer         │
└────────┬─────────┘         └────────▲─────────┘
         │                              │
         │ "Receive 50M in 90 days"     │ Lump sum / partial
         │ → 1 invoice                  │ remittance at maturity
         ▼                              │
┌──────────────────┐         ┌──────────────────┐
│  Global Investor │  buys   │  Seoul Café HQ    │
│  (USDC holder)   │  for    │  Self-guarantee   │
│   = Investor     │  48M    │   = Guarantor     │
└──────────────────┘          └──────────────────┘
```

**Key points:**

- **Coffee Farm (Seller)** closed the sale but **can't wait 90 days** → sells the receivable for 48M
- **Seoul Café Chain (Payer)** pays 50M at maturity (lump sum or split), behavior unchanged
- **Investor** recovers 50M after 90 days → ~2M profit ([[APR]]-equivalent)
- A common variant: **Seoul Café HQ becomes the Guarantor** of its franchise locations' receivables → strong credit signal → receivable trades at a tighter discount

| Actor | Real-world identity | Why they participate |
|-------|---------------------|----------------------|
| Coffee Farm | Vietnamese smallholder / co-op | Solves the **cash gap** (traditional factoring rejects them) |
| Seoul Café Chain | Korean franchise headquarters | Keeps its net-90 payment terms, nothing changes |
| Global Investor | Stablecoin holder, DeFi user | Wants **real-asset-backed yield** instead of DeFi-Ponzi yield |
| Seoul Café HQ (Guarantor) | Parent company with strong credit | Funds its franchises + earns guarantee fees |

---

### Telling Seller From Payer — A Cheat-Sheet

> [!tip] Three questions when you're unsure
> 1. **"Who holds the right to be paid?"** → That's the [[Seller]]. The one **who needs cash now**.
> 2. **"Who actually pays back (monthly or at maturity)?"** → That's the [[Payer]]. The debtor carrying `defaultPaymentMethod`.
> 3. **"Who pays cash up front and recovers later?"** → That's the [[Investor]].

| Question | Scenario A | Scenario B |
|----------|------------|------------|
| Who holds the receivable? | Mekong Motors | Coffee Farm |
| Who pays it back (monthly / at maturity)? | Sopheap (TADA driver) | Seoul Café Chain |
| Who fronts the cash? | Jihoon (Korean investor) | Global USDC investor |
| Possible Guarantor? | Angkor Capital / Mekong Motors itself / [[MVL]] (`isInternalMvl`) | Seoul Café HQ |
| `defaultPaymentMethod` | `TADA` (auto-deducted from earnings) | `FIAT_SELLER` (bank transfer) or `WALLET` |
| `invoiceCounts` | 12 (monthly) | 1 (bullet) or 3–6 |

> [!note] One person can wear two hats
> - **Seller = Guarantor**: Mekong Motors guarantees its own receivables — strong credit signal, tighter discount
> - **Operator = Guarantor**: [[MVL]] itself as guarantor (`isInternalMvl=true`); knows TADA drivers best, can price risk most accurately
> - But **Seller = Payer is impossible**: that would mean owing money to yourself

---

### Where the Guarantor Plugs In — Layering a Guarantor onto Scenario A

Jihoon asks: *"What if Sopheap quits TADA tomorrow? Or crashes the scooter? Where does my 2.8M go?"*

This is where the [[Guarantor]] enters as the **fourth party** in the deal.

```
        Without (uninsured)              With (guaranteed)
        ────────────────────             ──────────────────
   Default → Jihoon eats it all     Default → contract pulls
   APR ↑ (risk premium)                       from Vault up to
   Investors slow to commit                   coverageRatio
                                      APR ↓ (safety premium)
                                      Guarantor pockets the fee
                                      Investors rush in
```

**Three guarantor archetypes (Scenario A):**

1. **Professional guarantor** — Angkor Capital pre-funds USDC collateral into the [[Guarantor Vault]], `coverageRatio=80`, `fee=5`. On default, the contract auto-pulls up to 2.4M KRW from the Vault.
2. **Seller self-guarantee** — Mekong Motors = Seller + Guarantor. "I sold it, I guarantee it" is a powerful trust signal; the receivable sells at a premium.
3. **MVL self-guarantee (`isInternalMvl=true`)** — [[MVL]] as guarantor. Owns TADA driver data → best-in-class risk pricing. Earns guarantee fees + controls default risk via `defaultPaymentMethod=TADA` (auto-deduction).

> [!note] The Guarantor is an "insurer," and the fee is "performance-based on profit"
> Crucially, the guarantee fee is **not charged on face value, but as a % of the investor's profit**. The math and worked examples live in a separate section.

### Why the Operator Registers Participants — and the RWA Demand Behind It

The most common misunderstanding about [[AR-Service]] is "why does the operator have to register every [[Seller]] · [[Payer]] · [[Guarantor]] one by one?" The answer: **registration is not the system judging creditworthiness — it is the system embedding off-chain trust that humans already established into on-chain rails.** That is the essence of [[RWA]] (Real-World Asset) tokenization — contracts are not smart. A contract does not know whether an address truly represents a creditor, debtor, or guarantor. So humans verify first, and that verification is then encoded into a form the contract can enforce automatically (a whitelist).

> [!note] Core thesis
> Registration in AR-Service is a **3-stage trust transfer**: "credit judgment → DB record → on-chain Registry whitelist." The system does not assign credit scores. Humans do, and contracts enforce those decisions like checkpoints.

### Three-Stage Trust Transfer (Off-chain → DB → On-chain Registry)

```
┌──────────────────────────┐   ┌──────────────────────────┐   ┌──────────────────────────┐
│ 1. Off-chain trust       │   │ 2. System DB record       │   │ 3. On-chain Registry      │
│  (human judgment)         │   │  (AR-Service)             │   │  (Smart Contract)         │
├──────────────────────────┤──▶├──────────────────────────┤──▶├──────────────────────────┤
│ • Credit check / financials│ │ • CreatePayerDto validate │   │ • SellerRegistry          │
│ • KYC/AML diligence        │ │ • externalRef mapping     │   │ • PayerRegistry           │
│ • Legal contracts          │ │ • dataProvisionConsent    │   │ • GuarantorRegistry       │
│ • TADA driver history etc.│ │ • walletAddress validation│   │ • e.g. ID = DMPY-00001    │
└──────────────────────────┘   └──────────────────────────┘   └──────────────────────────┘
   (human decision)              (DB row created)            (whitelist = checkpoint armed)
```

Skip any stage and the deal does not exist. In particular, without stage 3 the [[Invoice]] cannot reach LISTED — i.e., it is not allowed onto the shelf.

### Key Fields Filled During Registration

The fields in `CreatePayerDto` (and the Seller/Guarantor DTOs) are not mere metadata — each carries **legal and operational weight**.

| Field | Meaning | Why it matters |
|-------|---------|----------------|
| `externalRef` | Real identifier in an external system (e.g., [[TADA]] driver ID, business registration number) | Maps the anonymous on-chain address to a real-world entity — the starting point for legal recovery on default |
| `dataProvisionConsent` | `RECEIVED` / `WAITING` / `REJECTED` | Legal basis for personal-data processing (KYC/AML). No contract proceeds without `RECEIVED` |
| `registeredBy` | `SELLER` / `OPERATOR` | Tracks who took responsibility — did a seller onboard their own counterparty, or did the operator vet them directly? |
| `defaultPaymentMethod` | `FIAT_SELLER` / `TADA` / `WALLET` / `GUARANTOR` | Declares how the debtor will repay — also embedded into the [[IPFS]] NFT metadata |
| `walletAddress` | EVM/BTC/Tezos/TON format validation | Terminal endpoint for on-chain funds. Format check blocks typos and wrong-chain addresses |
| `status` | `ACTIVE` / `PENDING` / `REJECTED` | Current trading eligibility. Operator can freeze a party after the fact |

> [!example] Actual registration flow (Payer example)
> 1. Cambodian [[TADA]] driver "Sophea" signs a 12-month installment for an electric motorbike
> 2. Operator inspects Sophea's TADA registration, ID, driving history → **off-chain credit decision**
> 3. DB record created: `externalRef = TADA-DRIVER-77231`, `dataProvisionConsent = RECEIVED`, `defaultPaymentMethod = TADA`, `walletAddress = 0x…`
> 4. Whitelisted on the on-chain `PayerRegistry` as e.g. `DMPY-00001`
> 5. From now on, the contract "knows" this address is a real debtor

### Five Concrete Benefits of Registration

Registration is not bureaucracy — it produces five specific benefits.

> [!tip] Registration = members-only club
> Not an open internet café anyone can walk into. A members-only club where the operator has verified each name on the list. Every deal inside the club is therefore "a deal between vetted members," not "a deal between anonymous addresses."

**① Fake-receivable blocking** — Without a registered Seller, no [[Invoice]] can be minted in the first place. This blocks the scenario of tokenizing fabricated receivables to defraud investors. Only real debts from real trades get funded.

**② Anti-impersonation (on-chain whitelist checkpoint)** — The Registry contract is an automated checkpoint. If an address not on the whitelist tries to call the contract as if it were a guarantor or debtor, the contract refuses. No human needs to police it — impersonation is blocked at the protocol layer.

**③ Regulatory compliance (KYC/AML)** — The `dataProvisionConsent` field is not a casual flag; it is a legal consent record. When financial regulators ask "did you know who the parties to this transaction were?" you can answer. This addresses the single biggest friction point at the intersection of emerging-market SME finance and crypto.

**④ Accountability** — `externalRef` ties the anonymous on-chain address back to a real-world legal person. On default, instead of "we have no idea whose wallet that is," you get "this wallet belongs to Mekong Motors, business reg. #XXX" — which feeds directly into legal collection and recovery procedures.

**⑤ Off-chain trust fused with on-chain automation** — The biggest picture. Humans are good at credit judgment (context, qualitative signals); contracts are good at moving funds, checking permissions, and enforcing rules (automatic, 24/7, transparent). Registration is the bridge. A human judges once, and that judgment becomes a permanent contract-level rule.

```
[Human judgment]   ─register─▶   [Contract auto-enforcement]
credit · KYC · contract           every tx gets auto-checked
(analog, one-time)                (digital, infinite repeat)
```

### Why There Is Demand for an RWA Invoice Market

The reason the registration system is engineered this carefully is that the [[RWA]] [[Invoice]] market it sits on top of has **strong two-sided demand**. You don't build this much infrastructure without demand to justify it.

#### Seller-side demand — preventing black-ink bankruptcy

The biggest driver is avoiding **black-ink bankruptcy** — going under while profitable on paper. SMEs (small/medium enterprises) collapse not because of losses but because receivables land 60–90 days after costs go out. Revenue is booked but cash dries up.

```
Jan: ₩100M in sales (only a right-to-be-paid)
Feb: ₩30M out for payroll + materials
Mar: same
Apr: Jan sales finally land ─────▶ but in between, cash ran out → black-ink bankruptcy
```

**Limits of existing solutions:**

| Solution | Problem |
|----------|---------|
| Bank loans | Emerging-market SMEs are rejected for insufficient credit |
| Traditional [[factoring]] | Slow (weeks to months), expensive, narrow eligibility |
| Trade finance | [[ADB]] estimates a **global trade-finance gap of ~$2.5T** — demand far exceeds supply |

**The RWA AR answer:** tokenize the receivable and sell it directly to global investors at a discount. Fast (days), cheap (no middlemen), open to anyone (including emerging-market SMEs).

#### Investor-side demand — thirst for real yield

Crypto has a massive [[stablecoin]] pool (hundreds of billions of USD as of 2026), and that capital **needs somewhere to earn**. But:

> [!warning] The DeFi Ponzi ceiling
> The 2020–2022 high-APR DeFi era was largely a Ponzi structure — paying token rewards by minting more tokens. Not real yield, just new entrants paying old entrants → [[Terra/Luna]] · [[FTX]] collapses shattered trust. Investors now ask "where does the yield actually come from?"

[[RWA]] AR answers that:
- **Real-asset-backed yield** — discount margin on actual corporate receivables. Source is identifiable.
- **Relative stability** — short-duration receivables are calmer than altcoin volatility
- **Short maturity** — 30–90 day cycles, capital turns over quickly, low lockup
- **Diversification** — each receivable is split via [[Fraction]] into ~₩100 units, spread across many
- **Small-ticket access** — traditional private debt has huge minimums; token slices welcome anyone

#### Why blockchain at all

Why does this have to live on a blockchain? Five concrete advantages:

| Advantage | Why |
|-----------|-----|
| **Borderless** | Cambodian receivable ↔ Korean investor, no bank routing |
| **24/7** | No market hours, no weekends |
| **Small-ticket** | [[Fraction]] tokenization enables ₩100-unit positions |
| **Transparency** | All transactions on-chain, repayment history auditable |
| **No middlemen** | No bank or broker fees skimming the spread |

#### Macro setup — why now

> [!example] Four macro drivers of the 2026 RWA market
> 1. **Stablecoin explosion** — USDC/USDT in the hundreds of billions, capital hunting yield
> 2. **Financial inclusion** — the first direct channel from emerging-market SMEs to global capital
> 3. **Regulatory clarity** — US/EU/Singapore frameworks for tokenized assets are crystallizing
> 4. **Institutional entry** — [[BlackRock]] · [[Franklin Templeton]] etc. launching tokenized RWA funds

#### Remaining risks and how this system answers them

Strong demand comes with strong risks. Every AR-Service design choice is an answer to one of these.

| Risk | This system's answer |
|------|----------------------|
| **Credit risk** (debtor default) | [[Guarantor]] · [[Guarantor Vault]] collateral coverage |
| **Off-chain trust dependency** | The 3-stage registration above + KYC + `externalRef` |
| **Liquidity** | [[Fraction]] tokenization enables secondary-market potential |
| **Regulatory uncertainty** | Compliance fields built in (`dataProvisionConsent`, `registeredBy`) |

> [!note] Summary
> Registration is elaborate because RWA demand is genuine. People truly want to buy and truly want to sell, so the trades need real safety rails. The operator's participant registration is the first such rail — without it, no amount of macro tailwind matters; fraud and default would unwind the market.

### How an Investment Product Is Created (Contract → Invoice → LISTED)

An investment product (= [[Invoice]] fractions) reaches the shelf via a straight-line pipeline: **Step 0 Registration → Step 1 Contract → Step 2 Invoice generation → Step 3 Confirmation → Step 4 IPFS → Step 5 On-chain register → Step 6 Guarantee register → Step 7 LISTED**. Each step has **prerequisites** and **lock/retry safeguards** before flowing into the next.

### Step 0 — Participant Registration (Registry)

The operator first stamps real-world trust into the system. Without this step, **the investment product literally cannot be created.**

| Registry | Who is registered | On-chain whitelist | Example ID |
|---|---|---|---|
| [[SellerRegistry]] | [[Seller]] (sells the receivable) | ✅ | DMSL-00001 |
| [[PayerRegistry]] | [[Payer]] (pays back) | ✅ | DMPY-00001 |
| [[GuarantorRegistry]] | (optional) [[Guarantor]] | ✅ | DMGR-00001 |
| (no registry) | [[Investor]] | ❌ (profile + JWT) | — |

> [!note] Registration is not just a DB insert — it includes an **on-chain REGISTER tx**. Only addresses written into the contract's whitelist are accepted by subsequent Invoice/Guarantee registrations.

`CreatePayerDto` key fields: `externalRef`, `dataProvisionConsent` (KYC consent), `registeredBy` (SELLER/OPERATOR), `defaultPaymentMethod` (FIAT_SELLER/TADA/WALLET/GUARANTOR), `walletAddress` (EVM/BTC/Tezos/TON format-validated), `status` (ACTIVE/PENDING/REJECTED).

---

### Step 1 — Create the Contract (the bungeoppang mold)

A [[Contract]] is **not an individual Invoice; it's the mold that stamps out N invoices (e.g. 12 monthly installments)** in one go. Build it carefully once, and `generateInvoices` will pop out all 12 at once.

`schema.prisma:301` key fields:

```ts
model Contract {
  sellerId                  String
  payerId                   String
  paymentAmountPerInstallment Decimal  // monthly payment (= each Invoice's faceValue)
  invoiceCounts             Int        // installments = number of Invoices to create
  startDate                 DateTime
  paymentDateMonthly        Int        // day-of-month (1~28)
  fractionCount             Int        // how many fractions per Invoice
  strategy                  PricingStrategy   // FIXED_APR / LINEAR_DECAY / STEP_DECAY
  strategyParams            Json              // {minApr, maxApr, decaySteps, ...}
  listingCurrency           String
  cutoffDays                Int               // how many days before maturity to stop selling
  status                    ContractStatus    // DRAFT → ...
  payerConfirmation         ConfirmationStatus  // WAITING/RECEIVED/REJECTED
}
```

> [!warning] A Contract starts in `DRAFT` and **cannot move forward until `payerConfirmation = RECEIVED`**. No invoices are generated without **the debtor's ([[Payer]]'s) prior consent.**

> [!example] Scenario A
> Mekong Motors (Seller) sells Sophea (Payer) a $3,000 e-motorbike on **12-month installment** →
> `paymentAmountPerInstallment = 250`, `invoiceCounts = 12`, `fractionCount = 10`, `strategy = LINEAR_DECAY`, `strategyParams = { minApr: 6, maxApr: 12 }`.

---

### Step 2 — generateInvoices (12 invoices at once)

`invoice-schedule-generator.service.ts::generateInvoices` reads the Contract and **bulk-inserts N invoices**.

```
Contract (12-month installment)
   │  generateInvoices()
   ▼
┌──────────────────────────────────────────────────────────┐
│ Invoice #1  dueDate = startDate + 1mo   APR = maxApr     │
│ Invoice #2  dueDate = startDate + 2mo   APR = (interp)   │
│ ...                                                      │
│ Invoice #12 dueDate = startDate + 12mo  APR = minApr     │
└──────────────────────────────────────────────────────────┘
   each: status = DRAFT, fractionCount/availableCount equal
```

#### APR interpolation (`spreadStrategyParams`)

> [!tip] **The longer to maturity, the higher the APR.** Invoice #1 (recovered quickly) gets maxApr; Invoice #12 (recovered last) gets minApr. This way investors can rationally trade off "duration vs yield."

| Strategy | Invoice #1 | Invoice #6 | Invoice #12 |
|---|---|---|---|
| `FIXED_APR(apr=7.5)` | 7.5% | 7.5% | 7.5% |
| `LINEAR_DECAY(max=12, min=6)` | 12% | 9% | 6% |
| `STEP_DECAY(max=12, min=6, decayRate, decaySteps)` | 12% | stepped ↓ | 6% |

#### Global lock

```ts
@RedisLock('invoice-generation')   // single global lock
async generateInvoices(contractId) { ... }
```

> [!warning] `@RedisLock('invoice-generation')` is **global, not per-Contract**. To guarantee invoiceId (starting at 100000, incrementing by 1) never collides, only one Contract's generate runs at a time across the entire system.

---

### Step 3 — confirmAllInvoices (DRAFT → CONFIRMED)

When the operator reviews all 12 and clicks OK, the following happens inside **a single `$transaction`**:

```
$transaction(async tx => {
  await tx.invoice.updateMany({ contractId, status: DRAFT })
                  .set({ status: CONFIRMED })

  await tx.contract.update({ status: INVOICE_GENERATED })

  for (invoice of invoices)
    await tx.outbox.create({ type: 'IPFS_UPLOAD', invoiceId, status: PENDING })
})
```

> [!note] If even one of the 12 fails, **the whole thing rolls back.** Prevents an inconsistent state where only some are queued for IPFS upload.

From this point, N `IPFS_UPLOAD` jobs sit in the [[Outbox]], picked up asynchronously by workers.

---

### Step 4 — IPFS Upload → ipfsCid (the receipt ticket)

[[invoice-ipfs.service.ts]] builds an NFT-metadata JSON (faceValue, dueDate, payer info, **defaultPaymentMethod**, …), uploads it to [[Pinata]], gets `ipfsCid`, and writes it onto the invoice row.

> [!warning] Without `ipfsCid`, **on-chain registration is refused.** The contract won't accept a "receivable" whose original certificate (= ipfsCid) is missing.

On success the Outbox entry is `IPFS_UPLOAD: COMPLETED`, and a follow-up `SC_REGISTER` job is enqueued as PENDING.

---

### Step 5 — SC_REGISTER (on-chain stamp)

A worker pulls `SC_REGISTER` from the OnchainTransaction queue and calls the invoice-register function on the [[MusubiARProtocol]] contract. APR and strategy are converted to **bps (×100)** and **enum integers (LINEAR_DECAY=0, FIXED_APR=1, STEP_DECAY=2)** (`invoice-onchain.service.ts:127`).

```
human 7.5%        →  on-chain bps 750
strategy=LINEAR_DECAY → 0
```

> [!warning] [[Outbox]] retry policy on failure: PENDING → PROCESSING → FAILED → … up to **5 retries**. After 5 failures: `PERMANENTLY_FAILED` + Slack alert. The safeguard that ensures money-related work never silently disappears.

On success, the invoice gets `onchainStatus = CONFIRMED` and `txHash`.

---

### Step 6 — Branch: register GuaranteePosition if guaranteed; otherwise straight to LISTED

> [!note] **This is where guaranteed vs unguaranteed listing paths diverge.** It's the most practical meaning of the [[isGuaranteed]] flag.

```
                  ┌──────────── isGuaranteed = false ────────────┐
                  │                                              │
   SC_REGISTER ───┤                                              ▼
   (success)      │                                          LISTED 🎉
                  │                                              ▲
                  └─ isGuaranteed = true ─┐                      │
                                          ▼                      │
                       GuaranteePosition on-chain register ──────┘
                       (bulk-register-guarantee-positions)
                       on success → Contract = GUARANTEE_REGISTERED
```

| Branch | Trigger location | LISTED condition |
|---|---|---|
| Unguaranteed | `invoice.message.controller.ts:181` | Immediately after SC_REGISTER success |
| Guaranteed | `guarantee-position.message.controller.ts:90` | After **all** invoices' GP register succeeds |

#### GuaranteePosition (model summary, `schema.prisma:407`)

```ts
model GuaranteePosition {
  invoiceId      Int      @unique          // 1 invoice : 1 GP
  guarantorId    String
  coverageRatio  Decimal(5,2)              // 0~100, loss-coverage cap (% of faceValue)
  fee            Decimal(5,2)              // >0~100, fee % on investor profit
  status         GuaranteePositionStatus
}
```

> [!tip] At the Contract level, only after **every** guaranteed invoice in that contract has its GP registered does the Contract advance to `GUARANTEE_REGISTERED`. That's a **separate contract-level "guarantee complete" signal.**

---

### Step 7 — LISTED (on the shelf)

```
status = LISTED
availableCount = fractionCount   (all fractions buyable)
```

Investors can now buy in the market. From here the state machine progresses as purchases happen.

---

### Full state flow — `InvoiceSavedStatus`

```
┌──────────┐  confirmAllInvoices   ┌────────────┐  SC_REGISTER + (GP)  ┌──────────┐
│  DRAFT   │ ───────────────────▶  │ CONFIRMED  │ ──────────────────▶  │  LISTED  │
└──────────┘     ($transaction)    └────────────┘                       └────┬─────┘
                                                                            │ first buy
                                                                            ▼
                                                                  ┌──────────────────┐
                                                                  │ PARTIALLY_FUNDED │
                                                                  └────┬─────────────┘
                                                                       │ last fraction sold
                                                                       ▼
                                                                  ┌──────────┐
                                                                  │ SOLD_OUT │
                                                                  └──────────┘
```

> [!note] **Sales status (InvoiceSavedStatus)** and **payment status (InvoicePaymentStatus: UNPAID → PARTIALLY_PAID/OVERDUE/MISSED → PAID/DEFAULTED)** are orthogonal. A `SOLD_OUT` invoice can still be `UNPAID`; at `LISTED` the [[Payer]] hasn't been asked to pay anything yet.

---

### Safeguards recap (why so many steps?)

| Safeguard | Where | Failure it prevents |
|---|---|---|
| `payerConfirmation = RECEIVED` | Contract DRAFT | Receivable created without debtor's knowledge |
| `@RedisLock('invoice-generation')` | generateInvoices | invoiceId collisions |
| `$transaction(confirmAllInvoices)` | Step 3 | Partial CONFIRMED |
| `ipfsCid` precheck | Just before Step 5 | Fake receivables without metadata |
| [[Outbox]] 5-retry + Slack | Steps 4·5·6 | Silent loss from transient failures |
| GP-all-or-nothing before LISTED | Step 6 → 7 | Under-guaranteed products hitting the shelf |

In short: **register participants (0) → mold (1) → bulk-generate 12 (2) → bulk-confirm (3) → certificate (4) → on-chain stamp (5) → (guarantee 6) → shelf (7)**. Every cell has a lock / transaction / retry / required field, so one misstep stalls at that step instead of leaking through to the next.

### Three Pricing Strategies and the Outbox

An [[invoice]] is a single note carrying a [[faceValue]] and a [[dueDate]] — "I'll give you 1,000원 in one month." But the price an investor pays should depend on **when they buy it**: someone buying with 30 days left and someone buying with 1 day left can't pay the same price. That's why [[Strategy|pricing strategies]] exist.

> [!note] The common principle — melting ice cream
> As time approaches `dueDate`, the **discount shrinks and price converges to faceValue**: `price(t) → faceValue as t → dueDate`. Even at the same APR, fewer remaining days means a smaller absolute discount. That is why buying late earns less.

### Strategy comparison

| Strategy | enum | strategyParams | Curve shape | Analogy |
|---|---|---|---|---|
| [[FIXED_APR]] | **1** | `{ apr }` | Flat single APR | Fixed price tag |
| [[LINEAR_DECAY]] | **0** | `{ maxApr, minApr }` | Straight-line decay | Slide |
| [[STEP_DECAY]] | **2** | `{ maxApr, minApr, decayRate, decaySteps }` | Drops at each tick | Staircase |

APR (= discount rate) over time:

```
APR
 │           FIXED_APR
 │  ─────────────────────────
 │
 │  \         LINEAR_DECAY
 │   \
 │    \
 │     \
 │      \____
 │
 │  ──┐       STEP_DECAY
 │    └──┐
 │       └──┐
 │          └──┐
 └─────────────────────────► t (purchase time)
   listing                  dueDate
```

`decaySteps` is a [[StepPeriod]] enum: `0=daily / 1=weekly / 2=bi-weekly / 3=monthly` — "how often does a step drop happen?" `decayRate` is the drop per step.

> [!example] LINEAR_DECAY worked example
> Suppose faceValue = 1,000,000원, 30-day maturity, `maxApr=12%`, `minApr=4%`.
> - At listing (t=0): APR ≈ 12% → large discount → cheap entry, higher yield
> - Near maturity (t=30): APR ≈ 4% → small discount → close to face value
> Same note, but earlier buyers are guaranteed a bigger profit.

### DB % ↔ on-chain bps conversion (×100)

The ar-service DB stores **human-readable percent** — `apr=7.5` literally means 7.5%. But [[Solidity]] contracts can't handle decimals, so values are transmitted as integer **basis points (bps)**. Just before going on-chain we multiply by **100**.

```
DB  : apr = 7.5        ( % )
SC  : apr = 750         ( bps,  1bp = 0.01% )
rule: bps = percent * 100
```

> [!warning] No-floating-point rule (around invoice-onchain.service.ts:127)
> All APR/fee values flow as **% in DB, bps on-chain**. If you see `apr.times(100)` in code, that is almost certainly this conversion. The reverse path (chain → DB) divides by 100. Forget the conversion on one side and you get a 100× error; centralize the conversion in a single helper.

Similarly, **the strategy enum is transmitted as a number, not a string**:

```ts
// on-chain payload encoding
LINEAR_DECAY = 0
FIXED_APR    = 1
STEP_DECAY   = 2
```

Guarantor-side percentages like `coverageRatio` and `fee` follow the same ×100 rule.

---

### The Outbox — making sure money work never silently disappears

The magic notebook ([[블록체인]]) sulks sometimes — RPCs die, gas spikes, nonces collide. But if a job like "register invoice" vanishes on one failure, the investment product itself never lists. So ar-service uses the [[Outbox 패턴|Outbox pattern]]: **persist every on-chain job to the DB first, then retry until it succeeds**. The table is called [[OnchainTransaction]].

```
[Service]                [DB: OnchainTransaction]              [Chain]
   │                            │                                 │
   │  1) insert(PENDING)        │                                 │
   ├───────────────────────────►│                                 │
   │                            │                                 │
   │  2) worker picks up        │                                 │
   │     markProcessing()       │                                 │
   │     PENDING → PROCESSING   │                                 │
   │◄───────────────────────────┤                                 │
   │                            │                                 │
   │  3) submit tx                                                │
   ├──────────────────────────────────────────────────────────────►│
   │                            │                                 │
   │  4-A) success → COMPLETED  │                                 │
   │  4-B) fail    → FAILED (retryCount++)                        │
   │  4-C) fail x5 → PERMANENTLY_FAILED ─► Slack alert             │
```

### State machine

| State | Meaning | Next |
|---|---|---|
| `PENDING` | Waiting; worker hasn't picked it up | → `PROCESSING` |
| `PROCESSING` | Worker has claimed it; **prevents duplicate execution** | → `COMPLETED` / `FAILED` |
| `COMPLETED` | On-chain success, `txHash` recorded | (done) |
| `FAILED` | Transient failure. retryCount++ and reset to `PENDING` | → `PENDING` (retry) |
| `PERMANENTLY_FAILED` | 5 failures; needs a human | (Slack alert) |

> [!tip] What `markProcessing()` does
> It performs the `PENDING → PROCESSING` transition **atomically**, so two workers can never pick up the same row. Once a worker holds it, no one else touches it until that worker dies or completes. It's effectively a per-job lock.

### Batch failure: split-into-singletons retry (isRevert)

When several entries are batched into one tx, a single bad item can revert the whole tx. Killing the innocent entries along with the bad one would be wasteful. So on `isRevert`, **`resetToPending` splits the batch back into individual jobs and re-queues them as PENDING** — and crucially, **does not increment retryCount**. The batch failure may not be that entry's fault, so it isn't penalized.

```
[Batch tx]  →  revert (because of one bad entry)
       │
       ▼
  isRevert detected
       │
       ▼
  resetToPending(each item)
       │  - status: PROCESSING → PENDING
       │  - retryCount: unchanged
       ▼
  retry one-by-one → only the truly bad one gets isolated
```

### Where it's used

Invoice registration (`SC_REGISTER`), on-chain metadata after IPFS upload, [[GuaranteePosition]] registration, payback sync (`paybackUsdc/paybackFiat`), `initiateRedeem` — **virtually every on-chain call that affects money or ownership flows through the Outbox**. The way ar-service "remembers what to do" *is* the outbox.

> [!warning] Why this matters
> From a user's point of view, "I bought but the invoice never listed" or "I paid but redeem doesn't work" are immediately financial incidents. Without the Outbox, one dropped RPC call would evaporate the job and require manual recovery. Five auto-retries plus a human-paging `PERMANENTLY_FAILED` + Slack alert on the sixth is what makes 24/7 unattended operation viable.

> [!note] One-line summary
> **Pricing strategy** is the rule that turns "when you buy" into a fair discount; **the Outbox** is the promise that "the result will reach the chain without ever being lost." Both are why [[ar-service]] can stay robust even though it never touches money directly.

### Guaranteed vs Non-guaranteed: Differences and Guarantee Fee Calculation

Every [[Invoice]] in [[AR-Service]] carries an `isGuaranteed` flag at issuance. That single boolean forks the **risk structure**, the **APR level**, the **listing path**, the **market exposure fields**, and even state transitions at the [[Contract]] level.

### 1) One-line definition

> [!note] What isGuaranteed means
> Whether or not there is a third party ([[Guarantor]]) that will cover investor losses if the [[Payer]] fails to repay at maturity. `true` if yes, `false` if no.

### 2) Risk vs APR trade-off

| Type | Risk bearer | Investor APR | Sales speed | Seller-side cost |
|---|---|---|---|---|
| **Non-guaranteed** (`isGuaranteed=false`) | Investor eats all losses | **Higher** (risk premium) | Possibly slower | No guarantee fee |
| **Guaranteed** (`isGuaranteed=true`) | [[Guarantor]] covers up to `coverageRatio` | **Lower** | Faster | `guaranteeFee` deducted |

Intuition: a guarantee is a form of insurance. The safer the deal, the lower the interest (APR) the investor earns — and part of that shaved-off yield flows to the guarantor.

### 3) Listing path divergence — branch at the code level

Both paths run through the same `confirmAllInvoices` → IPFS → [[SC Register]] pipeline, but the final step is different.

```
[Non-guaranteed path]
DRAFT → CONFIRMED → IPFS_UPLOAD → SC_REGISTER ──► LISTED ✅
                                  (invoice.message.controller.ts:181)

[Guaranteed path]
DRAFT → CONFIRMED → IPFS_UPLOAD → SC_REGISTER → (waiting)
                                                  │
                                                  ▼
                                  GuaranteePosition on-chain registration OK
                                  (guarantee-position.message.controller.ts:90)
                                                  │
                                                  ▼
                                                LISTED ✅
                                                  │
                                                  ▼
                          Once all guaranteed invoices have GPs registered:
                          Contract.status = GUARANTEE_REGISTERED
```

> [!warning] A guaranteed invoice never reaches LISTED until its GP is registered
> Meaning it doesn't show up on the [[Investor]] marketplace and cannot be purchased. The [[Guarantor Vault]] collateral must be ready and `GuaranteePosition` must be written to the on-chain Registry before the invoice goes live.

### 4) GuaranteePosition model

Per `schema.prisma:407` (invoiceId is `@unique` — exactly one guarantee per invoice):

```prisma
model GuaranteePosition {
  id            String
  invoiceId     Int      @unique
  guarantorId   String
  coverageRatio Decimal  @db.Decimal(5, 2)  // loss coverage cap %
  fee           Decimal  @db.Decimal(5, 2)  // guarantee fee % (profit-based)
  status        GuaranteePositionStatus
  ...
}
```

Two fields here that look similar are often confused. **You must hold them apart**.

> [!tip] fee vs coverageRatio — two %s with totally different meanings
> - **`coverageRatio`** = **loss coverage cap**, based on the **face value (faceValue)**
>   - e.g. faceValue 3,000,000 KRW, coverageRatio 80% → up to **2,400,000 KRW** drawn from the [[Guarantor Vault]] on default
> - **`fee`** = **the guarantor's earnings**, based on **investor profit (totalProfit)**
>   - e.g. profit 200,000 KRW, fee 5% → guarantee fee **10,000 KRW**
>   - Charged **only when the investor actually profits** (zero if profit ≤ 0)

### 5) Three flavors of guarantor — who actually backs the deal

> [!example] Scenario: [[TADA]] driver Sophea's 3,000,000 KRW, 12-month electric scooter installment
> Seller = Mekong Motors, Payer = Sophea, Investor = Jihoon. Three [[Guarantor]] patterns can plug in here.

**Case 1 — Professional guarantee institution (third-party guarantee)**
- e.g. Angkor Capital (a Cambodian credit guarantee firm)
- Deposits USDC/MVL collateral into the [[Guarantor Vault]] up front
- Guarantees `coverageRatio=80%`, charges `fee=5%`
- On default, drains up to 2.4M KRW from the Vault → earns a guarantee fee per deal
- The most conventional insurance model

**Case 2 — Seller self-guarantee**
- The seller (Mekong Motors) doubles as the guarantor
- A **trust signal** that says "I stand behind what I sell" → faster investor pickup → higher selling price on the receivable
- Also a marketing tool to show confidence in their own portfolio

**Case 3 — MVL internal guarantee (`isInternalMvl=true`)**
- [[MVL]] itself takes the guarantor role
- MVL has the **deepest data** on [[TADA]] ecosystem drivers (rides, income, reputation)
- If `defaultPaymentMethod=TADA`, repayment is auto-deducted from driver income, so recovery risk is also controllable
- Three birds with one stone: guarantee fee revenue + boosted core (TADA) revenue + capital attracted to the platform

### 6) The guaranteeFee formula

Per `purchase-fee.service.ts:75`:

```
guaranteeFee = totalProfit × (fee% / 100)        // only when totalProfit > 0
            = 0                                  // when totalProfit ≤ 0
```

where:
```
totalProfit = faceValue - purchasePrice   (face value minus what the investor paid)
```

> [!warning] It's a % of **profit**, not of face value
> Contrary to intuition, the guarantee fee is **not** `faceValue × fee`. It's only taken from the investor's actual gain (`totalProfit`). For late-stage purchases where the price has crept up close to face value, the guarantee fee converges to near zero.

Allowed `fee` range (`bulk-register-guarantee-positions.usecase.ts:414`):
- `0 < fee ≤ 100` (%)
- Up to 2 decimal places (Decimal(5,2))
- **Negotiated per deal** — no fixed default

### 7) Worked example — face value 3,000,000 KRW, purchase price 2,800,000 KRW

```
faceValue         = 3,000,000 KRW
purchasePrice     = 2,800,000 KRW
totalProfit       =   200,000 KRW   ← basis for the guarantee fee
```

Sweeping `fee%`:

| fee % | guaranteeFee formula | guaranteeFee |
|---|---|---|
| 3% | 200,000 × 3% | **6,000 KRW** |
| **5%** | **200,000 × 5%** | **10,000 KRW** |
| 10% | 200,000 × 10% | 20,000 KRW |
| 20% | 200,000 × 20% | 40,000 KRW |

Following the full settlement (with `platformFee` currently at 0%):

```
net redeem    = faceValue − guaranteeFee − platformFee
              = 3,000,000 − 10,000 − 0
              = 2,990,000 KRW

investor PnL  = net redeem − purchasePrice
              = 2,990,000 − 2,800,000
              = 190,000 KRW   ← expectedReturn
```

So adding a guarantee shrinks investor profit from 200K to **190K KRW (a 10K cut)**, and that 10K becomes the [[Guarantor]]'s take. For 10K the investor effectively bought insurance worth up to 2.4M KRW of default coverage.

### 8) Market exposure field differences

Finally, the marketplace API responses diverge too.

| Field | Non-guaranteed | Guaranteed |
|---|---|---|
| `isGuaranteed` | `false` | `true` |
| `coverageRatio` | `null` | e.g. `80.00` |
| `guaranteeFee` (expected) | `null` | computed value shown |
| `guarantor` info | `null` | guarantor id/name shown |

Investors use this metadata to distinguish — at the same headline [[APR]] — between "a safe 5% with backing" and "a risky 5% without it."

### Deep dive into the Guarantor Vault

The [[Guarantor Vault]] is the on-chain safe where a [[Guarantor]] pre-locks USDC/MVL collateral to back their "I'll cover the default" promise. [[ar-service]] does **not** custody this money — the contract does. ar-service's role is purely **bookkeeping and surveillance**: track balances, audit movements, and shout when collateral runs thin.

### 1) Why the Vault exists

> [!note] Core question — is a promise enough?
> No. If a guarantor vanishes or has no funds, the [[Investor]] takes the loss directly. So every [[Guaranteed]] invoice must have its guarantor lock USDC/MVL into the Vault **before GuaranteePosition registration**, and ar-service starts checking sufficiency **7 days before maturity**.

```
Guarantor promise ─┐
                   ├─► Vault (USDC/MVL locked) ─► contract pulls on default
ar-service ────────┘   monitors & records, never custodies
```

| Token   | Valuation                                                     | Note                                 |
| ------- | ------------------------------------------------------------- | ------------------------------------ |
| USDC    | 1:1 as USD                                                    | No volatility                        |
| BASE_MVL / MVL | On-chain call `totalUSDValue(bytes32 guarantorId)`     | ar-service refuses to price MVL itself |

> [!tip] MVL→USD conversion always goes through [[guarantor-vault-contract.service.ts]]:30's `vaultContract.read.totalUSDValue()`, returning USDC at 6 decimals. ar-service stores only raw amounts (`Decimal(30,18)`) in DB and relies 100% on the contract's live valuation — because MVL is volatile, any cached price would be stale within minutes.

### 2) Double ledger — Transaction (history) + Balance (snapshot)

```
┌────────────────────────────────────────┐  ┌────────────────────────────────┐
│ GuarantorVaultTransaction (history)    │  │ GuarantorVaultBalance (live)   │
│  txHash @unique                        │  │  @@unique[guarantorId,token,   │
│  type: DEPOSIT / WITHDRAWAL            │  │           network]             │
│  amount Decimal(30,18)                 │  │  amount Decimal(30,18)         │
│  status: PENDING → CONFIRMED           │  │  (+/- on confirmation)         │
└────────────────────────────────────────┘  └────────────────────────────────┘
                       │                                      ▲
                       └─── single $transaction on confirm ───┘
```

> [!example] Why split the books?
> - **Transaction** is the *immutable audit trail*: when, which tx hash, how much.
> - **Balance** is the *aggregated cache* for "how much USDC/MVL does this guarantor hold right now?"
> Both update inside the same `$transaction`, so history and balance can never drift apart even for a moment.

### 3) Deposit/withdrawal confirmation — Path A vs Path B

> [!note] Vault deposits get recorded two ways — pre-logged by a human and later confirmed by the on-chain event, or auto-matched when the event arrives with no prior record.

**Path A — recordDeposit (pre-log, then confirm)**

```
1. admin (admin-service) → recordDeposit(guarantorId, txHash, amount)
2. GuarantorVaultTransaction row created (status=PENDING)
3. watcher-worker spots the tx on-chain → TRANSACTION_CREATED_V2 event
4. ar-service confirm-vault-transaction.usecase picks it up →
   PENDING → CONFIRMED + Balance += amount  (one $transaction)
```

**Path B — autoConfirmVaultTx (auto-detect)**

```
1. (no admin pre-log)
2. watcher-worker sees a transfer to/from the Vault contract
3. transfer.from or transfer.to matched against registered guarantor wallets
4. Transaction row inserted directly as CONFIRMED + Balance updated
```

> [!tip] Both exist because operators forget to pre-log, and guarantors sometimes deposit directly from their own wallet — Path B catches those. When a pre-log exists, Path A carries richer metadata (admin who initiated, intent, etc.).

### 4) On-chain amount wins + patchConfirmedTx reconciliation

> [!warning] The human-entered `amount` and the on-chain amount can disagree.
> Admin pre-logs "10,000 USDC incoming" but the chain shows 9,998 (fee, slippage, miskey). Which one is truth?

[[confirm-vault-transaction.usecase.ts]]:124's verdict:

```
on-chain amount (raw transfer event value) ALWAYS WINS

if (pending.amount !== onchain.amount) {
    patchConfirmedTx(txHash, onchain.amount)   // rewrite Transaction.amount
    Balance += onchain.amount                  // Balance follows on-chain
}
// both lines run inside a single prisma.$transaction → atomic
```

| Source           | Authority         | Behavior                            |
| ---------------- | ----------------- | ----------------------------------- |
| Admin pre-log    | Advisory          | Overwritten if it disagrees with chain |
| watcher event    | **Single truth**  | Drives both Balance and Transaction |

### 5) 7-day proactive collateral sufficiency cron

> [!example] [[CRON_AR_SERVICE_CHECK_GUARANTOR_COLLATERAL]] — the daily inspector
> [[check-guarantor-collateral.usecase.ts]] runs daily. It gathers **every guaranteed invoice maturing in the next 7 days**, sums the guarantor's total liability, and compares against the live Vault valuation.

```
for each active guarantor:
    liability = Σ (invoice.faceValue × invoice.coverageRatio / 100)
                where invoice.dueDate within [today, today + 7d]
                and  invoice.guarantorId == this guarantor
                and  invoice.paymentStatus != PAID
    vaultValue = vaultContract.read.totalUSDValue(guarantorId)   # USDC 6 decimals

    if vaultValue < liability:
        send email warning  (operators + guarantor)
        # 7-day head start gives time to top up
```

| Quantity                  | Formula                                    |
| ------------------------- | ------------------------------------------ |
| Single-invoice liability  | `faceValue × coverageRatio / 100`          |
| Guarantor total liability | Sum the above over invoices due within 7d  |
| Vault valuation           | `totalUSDValue(guarantorId)` on-chain read |
| Alert condition           | `vaultValue < liability`                   |

> [!tip] The 7-day window is the [[lead time]] needed for "detect → notify guarantor → top up → on-chain confirmation" in business days.

### 6) MonitorVaultMovementUc — thief watch

> [!note] [[MonitorVaultMovementUc]] watches USDC movement across ar-service's [[escrow trio]].

```
Monitored (infra.config.ts:330, monitored:true):
  - Guarantor Vault
  - Payback Escrow
  - Purchase Escrow
   (AR Protocol contract is monitored:false — internal protocol moves create signal noise)

Token: USDC only
   (MVL is excluded — price volatility means legitimate moves can look like huge amounts)
```

Counterparty classification — the other wallet in the transfer gets labeled:

| Label             | Match rule                                       | Action                       |
| ----------------- | ------------------------------------------------ | ---------------------------- |
| **known**         | Registered guarantor / investor wallet           | Log only, no alert           |
| **internal**      | AR Protocol contract addresses (trio + protocol) | Log only, no alert           |
| **SUSPICIOUS**    | None of the above                                | `@channel` Slack alert fires |

> [!warning] A SUSPICIOUS hit pings the entire ops team via `@channel`. Best case it's a missing whitelist entry; worst case it's an actual exfiltration in progress.

### 7) So why does the Vault run short?

> [!warning] Three scenarios trip the 7-day cron even though collateral is locked.

**Cause 1 — MVL price crash**

```
T-30d: 1 MVL = $0.10, guarantor's 100,000 MVL = $10,000 (sufficient)
T-1d:  1 MVL = $0.03, guarantor's 100,000 MVL = $3,000  (short!)
       ↑ ar-service polls totalUSDValue live → detects instantly
```

**Cause 2 — Shared pool (one guarantor backing many invoices)**

```
Guarantor A's Vault: $50,000 USDC

  Invoice #1 (matures D+3, liability $20,000) ┐
  Invoice #2 (matures D+5, liability $20,000) ├─ total liability $70,000
  Invoice #3 (matures D+6, liability $30,000) ┘   > Vault $50,000 → short
```

The Vault is a **global pool**, not partitioned per-invoice. Every new GuaranteePosition adds to the cumulative liability against the same balance.

**Cause 3 — Guarantor withdrawal**

```
Mid-flight guarantee → guarantor pulls a partial withdrawal (WITHDRAWAL tx)
  → Balance drops → may no longer cover the other invoices it implicitly backed
```

> [!tip] Common thread: **liability is locked-in but vaultValue floats**. So ar-service watches the *delta* between the two daily, not the absolute balance. The 7-day proactive window exists precisely so the guarantor has business days to close that gap before maturity hits.

### Repayment & Compensation on Default

This is the stage that makes [[ar-service]]'s "never directly touches money" principle the most visible. Repayment is a four-actor relay, and when default strikes, compensation is executed entirely by the [[on-chain]] contract.

### 1. Repayment — Four Actors

```
┌─────────────────┐   real transfer    ┌──────────────────────┐
│ ① Payer         │ ───────────────▶  │ Bank acct / Payback   │
│ (debtor)        │   (off-chain or   │ Escrow (on-chain)     │
└─────────────────┘    on-chain)      └──────────────────────┘
        │                                       ▲
        │  "I paid"                              │  watcher observes
        ▼                                       │
┌─────────────────┐   @TrustedController        │
│ ② Admin         │ ─────────────────────────┐ │
│ (admin-service) │   records PaymentEvent    │ │
└─────────────────┘                          │ │
        │                                    ▼ ▼
        ▼                            ┌──────────────────────┐
┌─────────────────┐   aggregator     │ ③ ar-service         │
│ DB              │ ◀───────────────│  (aggregate + sync)   │
│ PaymentEvent    │                  └──────────────────────┘
└─────────────────┘                              │
                                                 │ paybackUsdc / paybackFiat
                                                 │ delta only
                                                 ▼
                                       ┌──────────────────────┐
                                       │ ④ AR Protocol        │
                                       │  Contract            │
                                       │  (Payback Escrow)    │
                                       └──────────────────────┘
```

| # | Actor | Job | Notes |
|---|---|---|---|
| ① | [[Payer]] | Sends real money | Bank account if cash, [[Payback Escrow]] if USDC |
| ② | [[Admin]] (admin-service) | Records `PaymentEvent` manually | `@TrustedController`, channel = `ADMIN_MANUAL` |
| ③ | [[ar-service]] | Aggregate + [[on-chain sync]] | `paybackUsdc` / `paybackFiat`, delta only |
| ④ | [[AR Protocol Contract]] | Holds repayment, pays on redeem | From Payback Escrow balance |

> [!note] Core idea
> [[ar-service]] is the **notebook clerk**. The truth of money lives in ① Payer's transfer and ④ the contract's Escrow balance; ②③ only *record and synchronize* the two.

#### PaymentEvent types Admin can record

- `PAID` — normal inflow
- `MISSED` — overdue with no payment
- `DEFAULTED` — judged irrecoverable

> [!warning] `OVERDUE` is NOT manually recorded
> `OVERDUE` is derived automatically by an ar-service cron comparing `dueDate` with now. Admin never stamps it directly.

Endpoints:

- Single: `recordManualPaymentEvent(...)`
- Bulk: `uploadPaymentEventsCsv(...)` — drop a bank statement copy, transcribe in one go

#### ar-service aggregation + sync (delta-only)

`invoice-payment-aggregator.service.ts` rolls `PaymentEvent`s per invoice into `totalPaid` / `paymentStatus`. Then `payment-sync-onchain.service.ts` pushes only the *increment*.

```ts
// pseudo
const syncableTotal = aggregateForSync(invoiceId)   // cumulative target
const delta = syncableTotal - lastSyncedAmount      // increment since last sync

if (delta < 0n) throw new Error('negative delta — payback cannot decrease')
if (delta === 0n) return                            // nothing to send

await arProtocol.paybackUsdc(invoiceId, delta)      // or paybackFiat
```

> [!warning] No negative deltas
> The contract's `payback` is **monotonically increasing**. Even if accounting needs a downward correction, ar-service never sends a negative sync; a guard throws before the call. Every correction must be settled in DB *before* the contract sees it.

> [!tip] @RedisLock serializes per invoice
> The aggregator wraps each invoice in an `@RedisLock`, so concurrent CSV uploads + auto crons can't race. paybackUsdc calls for one invoice line up single-file.

### 2. Default Verdict & initiateRedeem

#### 2-1. Default verdict — `hasDefault` wins

`deriveStatus()` priority:

```
hasDefault → DEFAULTED  (★ highest — beats anything)
fullyPaid  → PAID
partiallyPaid → PARTIALLY_PAID
overdue    → OVERDUE / MISSED
else       → UNPAID
```

> [!note] DEFAULTED is sticky
> Once Admin stamps a single `DEFAULTED` PaymentEvent on an invoice, no subsequent `PAID` event can flip it back. It's the trigger for guarantor execution — so we lock it conservatively.

#### 2-2. Redeem window opens by **maturity only**

`findMaturedAwaitingInitiateRedeem` (invoice.repository.ts:98) requires:

| Field | Condition |
|---|---|
| `onchainStatus` | `CONFIRMED` |
| `dueDate` | `<= today + 1 day` |
| `status` | `PARTIALLY_FUNDED` or `SOLD_OUT` |

> [!warning] No `paymentStatus = PAID` requirement
> The window opens on **maturity**, not on "the money is in." Even a fully-defaulted invoice triggers `initiateRedeem` once mature. What investors actually receive equals *whatever is in Escrow and Vault at that moment*.

#### 2-3. initiateRedeem call

`process-initiate-redeem.usecase.ts` picks invoices meeting the conditions and calls `arProtocolContract.initiateRedeem(invoiceId)`. From here, investors can redeem their fractions.

### 3. FractionsRedeemed event — the truth of the split

When an investor sends a redeem tx, the contract emits `FractionsRedeemed`. [[watcher-worker]] catches it and feeds ar-service. Decoded payload:

| Field | Meaning |
|---|---|
| `redemptionAmountUsdc` | What the investor actually received (USDC) |
| `protocolFeeUsdc` | Protocol/MVL cut |
| `guarantorFeeUsdc` | Guarantor's fee (if any) |

> [!note] guarantorFee > 0 means the contract paid the guarantor in the same settlement
> ar-service just records the three numbers. There is **no code path** that says "now send X to the guarantor" — the contract settles investor / protocol / guarantor atomically in one tx.

Final invoice state by redeem outcome:

| Case | Invoice status |
|---|---|
| Full face value recovered | `REDEEMED` |
| Partial recovery (default + partial guarantor coverage) | `PARTIAL_REDEEM` |

### 4. Payback Escrow short → pull from Vault up to coverageRatio

The contract decides priority:

```
redeem request
    │
    ▼
┌──────────────────────────────────┐
│ Payback Escrow has enough?       │
└──────────────────────────────────┘
    │ Yes                       │ No
    ▼                            ▼
[Pay from Payback Escrow]   ┌──────────────────────────────────┐
                            │ Guaranteed? (isGuaranteed)        │
                            └──────────────────────────────────┘
                                │ Yes                   │ No
                                ▼                        ▼
                    [Pull from Guarantor Vault    [Shortfall = investor loss]
                     up to coverageRatio]          (the price of un-guaranteed)
```

> [!example] Default compensation walkthrough
> - faceValue 3,000,000, coverageRatio 80%, fee 5%
> - At maturity, Payback Escrow balance: 1,000,000 (Payer only partially paid)
> - Shortfall 2,000,000 → Guarantor Vault can supply up to `3,000,000 × 80% = 2,400,000` → pulls 2,000,000
> - Investor gets redemptionAmount; FractionsRedeemed carries non-zero guarantorFee to settle the guarantor
> - Invoice status: `REDEEMED` if face value is fully met, else `PARTIAL_REDEEM`

> [!warning] Un-guaranteed = no safety net
> If `isGuaranteed = false`, the Vault-pull branch doesn't even exist. Only what landed in Payback Escrow is distributed pro-rata, and the rest is investor loss. That's why the APR was higher.

### 5. MVL collateral USD valuation — delegated on-chain

The Vault holds USDC *and* MVL as collateral. MVL is volatile, and [[ar-service]] does **not** price it itself.

```ts
// guarantor-vault-contract.service.ts:30
const usd = await vaultContract.read.totalUSDValue(guarantorIdAsBytes32)
// → returned in USDC 6-decimal units
```

> [!note] Valuation lives on-chain
> One call to `totalUSDValue(bytes32 guarantorId)` and you're done. MVL → USD conversion, oracle dependency, rounding rules — all inside the contract. ar-service only knows token identifiers (`BASE_MVL` / `MVL`) and stores **raw amounts** in DB.

Why delegate:

- **Volatility containment**: even during a price crash, ar-service and the contract can't disagree on the value — they read the same source
- **Audit simplification**: [[MonitorVaultMovementUc]] can stay USDC-only (no MVL pricing responsibility)
- **Proactive monitoring** works: `check-guarantor-collateral.usecase.ts` runs daily, compares `faceValue × coverageRatio` (liability) vs `totalUSDValue` for guaranteed invoices maturing in the next 7 days, and emails the guarantor when short — *before* an actual default

> [!tip] ar-service's responsibility boundary
> - **Record**: PaymentEvent (stamped by Admin)
> - **Aggregate**: compute totalPaid
> - **Sync**: push delta to contract (no negatives)
> - **Coordinate**: trigger initiateRedeem, decode FractionsRedeemed, flip invoice status
> - **Delegate**: compensation math, money movement, MVL→USD valuation — all to the contract

### The Three On-Chain Vaults and When the Redeem Window Opens

### MVL's "vending machines," not wallets

Kill the biggest misconception first. The [[ar-service]] codebase contains **zero** references to a wallet called `treasury`. There is no "MVL company account" that holds investor money.

Instead, funds are locked inside **three on-chain smart contracts**, each with its own rules — money only flows out when the rules match. So the [[금고 3형제|Three Vaults]] aren't wallets so much as **vending machines**. Drop a coin in, but if the right button isn't pressed (maturity, status, signature), no soda comes out.

> [!note] Core mental model
> [[ar-service]] is a **bookkeeping clerk** that never touches money directly. The real custodians are three smart contracts: [[Guarantor Vault]] / [[Purchase Escrow]] / [[Payback Escrow]]. ar-service mirrors their movements into its DB and blows the whistle on Slack when something looks SUSPICIOUS.

### Role separation table

| Vault | Whose money? | Inflow → Outflow | Code location |
|---|---|---|---|
| **[[Guarantor Vault]]** | Guarantor's **collateral** (USDC + MVL) | Guarantor pre-deposits → on default, contract pulls [[coverageRatio]] portion into [[Payback Escrow]] | `ArContractConfig` (`infra.config.ts:330`), `monitored:true` |
| **[[Purchase Escrow]]** | **Investor purchase funds** | Investor → Escrow → (pre-maturity) seller settlement | `monitored:true` |
| **[[Payback Escrow]]** | Payer's **repayment** (USDC) | Payer → Escrow → post-maturity investor redeem | `monitored:true` |
| (ref) AR Protocol | Holds **no** funds — only the rules (vending-machine brain) | `initiateRedeem`, `FractionsRedeemed`, etc. | `monitored:false` |

```
         ┌────────────────┐                       ┌────────────────┐
guarantor│ Guarantor Vault│ ─(on default pull)──▶ │                │
         └────────────────┘                       │                │
                                                  │ Payback Escrow │ ──(redeem)──▶ investor
         ┌────────────────┐    (payer repays)    │                │
 payer ─────────────────────────────────────────▶ └────────────────┘
                                                  
         ┌────────────────┐    (early settlement)
investor │ Purchase Escrow│ ─────────────────────────────────────▶ seller
         └────────────────┘
```

> [!tip] "So who watches the vending machines?" — **watcher-worker = the chain's CCTV**
> ar-service does not poll the chain itself. An external [[watcher-worker]] scans blocks via RPC, and when it sees fund movement at any of the three vault addresses, it emits a **`TRANSACTION_CREATED_V2`** event. ar-service then:
> 1. Runs `MonitorVaultMovementUc` (registered guarantor/investor=normal, AR protocol contract=internal, unknown wallet=**SUSPICIOUS → @channel Slack**).
> 2. Calls `autoConfirmVaultTx` to flip Guarantor Vault tx records from `PENDING → CONFIRMED`.
>
> The contract does not watch anything — **the watcher-worker watches the contracts**, and ar-service listens to the watcher.

### The less-obvious role of Purchase Escrow — seller settlement

[[Purchase Escrow]] isn't just a holding pen for investor cash. It's also the **window the seller uses to get cash before maturity**. This is the whole point of the [[RWA 인보이스 투자 상품|RWA invoice product]]: the seller needs cash *now*, not in 60–90 days, so they sell the receivable at a discount.

```
1) Investor buys fractions          → cash lands in Purchase Escrow
2) Seller settlement                → Purchase Escrow → seller wallet/account
3) (separately) payer repays at due → cash lands in Payback Escrow
4) Investor redeems                 → Payback Escrow → investor
```

> [!example] Purchase Escrow vs Payback Escrow — the one-liner that sticks
> - **[[Purchase Escrow]]** = investor cash → seller (front-end settlement)
> - **[[Payback Escrow]]** = payer cash → investor (back-end settlement)
>
> Separating them is what lets "the payer hasn't paid yet, but the seller already got cash" coexist as a normal state.

### The real condition for opening the redeem window — `findMaturedAwaitingInitiateRedeem`

Common wrong mental model: "Once the payer pays and `paymentStatus = PAID`, the investor can redeem." **Wrong.** The redeem window opens on **maturity**, not on payment status.

The code (`invoice.repository.ts:98` — `findMaturedAwaitingInitiateRedeem`) ANDs exactly these conditions:

```ts
// pseudocode
WHERE onchainStatus = CONFIRMED
  AND dueDate <= today + 1day      // maturity reached (or imminent)
  AND status IN (PARTIALLY_FUNDED, SOLD_OUT)
// note: NO check on paymentStatus = PAID
```

| Common misread | Reality |
|---|---|
| Need `paymentStatus = PAID` to redeem | ❌ Not in the WHERE clause |
| `paymentStatus` is the switch | ❌ Maturity is the switch |
| No repayment → redeem is blocked | ❌ Window opens. You just receive **only what's in the vault** |

> [!warning] Window open ≠ vault full
> Once `findMaturedAwaitingInitiateRedeem` picks an invoice and calls `arProtocolContract.initiateRedeem`, the AR Protocol contract emits `FractionsRedeemed` and settles. The investor receives **whatever amount is actually sitting in the vault at that moment** — the face value is not auto-guaranteed.

### What the investor actually receives — "as much as the vault holds"

| Situation | Vault paying out | Outcome |
|---|---|---|
| Payer pays in full by maturity | [[Payback Escrow]] | `REDEEMED` — full face value (minus fees) |
| Payer pays partially | [[Payback Escrow]] (short) | `PARTIAL_REDEEM` — only what came in |
| Default + **guaranteed** invoice | [[Payback Escrow]] (partial) + **[[Guarantor Vault]]** (collateral up to [[coverageRatio]]) | Contract auto-pulls collateral to top up |
| Default + **non-guaranteed** invoice | (none) | Investor eats the loss |

Decode the [[FractionsRedeemed]] event and you get `redemptionAmountUsdc / protocolFeeUsdc / guarantorFeeUsdc`. **A non-zero `guarantorFeeUsdc` is the signal that the guarantor was involved** in that settlement (collateral was pulled, or a guarantor fee was deducted).

### Putting the flow together — payment events are "refueling," not a switch

```
[Payer actually sends money] ─────▶ USDC accumulates in Payback Escrow
       │
       ▼
[Admin records PaymentEvent via
 admin-service]              ─────▶ ar-service syncs paybackUsdc/Fiat to the contract
                                    (delta only; never negative)

[Due date reached] ───────────────▶ findMaturedAwaitingInitiateRedeem picks it up
                                    → arProtocolContract.initiateRedeem
                                    → FractionsRedeemed event
                                    → investor can redeem
```

So recording a [[PaymentEvent]] is **not** the switch that decides redeem-or-not — it's **the act of refueling the vault (mainly Payback Escrow) and announcing that fact on-chain**. The calendar decides whether the window opens; the vault balance decides how much comes out.

> [!note] One-liner
> There is no `treasury` in [[ar-service]]. Funds live in three vending-machine contracts — [[Guarantor Vault]] (collateral) · [[Purchase Escrow]] (investor purchase + seller early settlement) · [[Payback Escrow]] (payer repayment + investor redeem) — and a [[watcher-worker]] relays their movements via `TRANSACTION_CREATED_V2`. The redeem window opens on **maturity**, not on `paymentStatus=PAID`, and the investor receives **as much as the vault holds at that moment**.

### MVL's Revenue Model and the 4 `defaultPaymentMethod` Values

This section covers two things together:
(1) what the four `defaultPaymentMethod` values stamped onto an [[invoice]] mean and **how the system actually uses them**, and
(2) how MVL actually makes money on top of [[ar-service]] — sharply separating what's **verified in code** from what's **inferred from the business model**.

---

### 1. The 4 `defaultPaymentMethod` Values — Metadata, Not Logic

First, let's clear up the most common misconception.

> [!warning] `defaultPaymentMethod` is NOT a branching condition
> Nowhere in [[ar-service]] does the code branch on this field like `if (defaultPaymentMethod === 'TADA')`. It is a **descriptive label** that gets baked into the [[IPFS]] NFT metadata's `attributes` array when an [[invoice]] is uploaded (`invoice-ipfs.service.ts:154`). Its purpose is to let an investor read the marketplace and understand "how is this debtor expected to repay" — nothing more.

Actual repayment runs through a totally separate flow: the debtor sends real money (outside [[ar-service]]), the operator records a `PaymentEvent` through [[admin-service]], and [[ar-service]] aggregates and syncs [[paybackUsdc]]/[[paybackFiat]] to the magic ledger (contract). `defaultPaymentMethod` does **not** automatically participate in any of that.

#### What the 4 values mean

| Value | Who repays | Where it flows | Real-world example |
|---|---|---|---|
| `FIAT_SELLER` | The [[지급인 Payer]] pays the [[판매자 Seller]] **in cash / bank transfer** | Off-chain → seller receives → seller reflects in system | A Cambodian café chain wiring monthly to a coffee farm |
| `TADA` | Auto-deducted from the [[지급인 Payer]]'s [[TADA]] app earnings | TADA settlement → embedded-finance deduction → repayment | A Phnom Penh [[TADA]] driver paying off an EV motorbike from ride earnings |
| `WALLET` | The [[지급인 Payer]] sends **on-chain directly** from their own crypto wallet | USDC transfer to the [[Payback Escrow]] contract | A crypto-native debtor sending USDC directly |
| `GUARANTOR` | A case where the [[보증인 Guarantor]] is expected to **pay on the debtor's behalf** from day one | Guarantor funds flow into the [[Payback Escrow]] | Debtor's repayment intent is weak; the guarantor effectively carries it |

> [!note] The interesting one is `TADA`
> `TADA` isn't just a payment method — it's **MVL's own Southeast Asian ride-hailing super-app**. MVL sells EV bikes / rental fees / etc. to Cambodian [[TADA]] drivers on installment, and deducts daily/weekly from their ride earnings — pure embedded finance. This is the smoking gun that [[ar-service]] isn't a generic RWA platform — it's a **financial layer wired directly into MVL's core business**.

#### Why is this metadata-only?

```
  [Invoice created]
        │
        ▼
  defaultPaymentMethod = "TADA"   ← from DB (Payer registration)
        │
        ▼
  invoice-ipfs.service.ts:154
        │
        ▼
  {
    "attributes": [
      { "trait_type": "defaultPaymentMethod", "value": "TADA" },
      ...
    ]
  }
        │
        ▼
   Upload to IPFS → ipfsCid issued
        │
        ▼
   Permanently recorded as on-chain NFT metadata
```

> [!tip] Why bake it into NFT attributes?
> The receivable trades as NFT [[조각 fraction]] tokens, and **each fraction needs to carry its own "this receivable is TADA-backed" label** so investors on a future secondary market can still assess risk. Humans (operators) handle the branching; the NFT carries the trust label.

---

### 2. MVL's Revenue Model — Verified in Code vs. Inferred

Now the main question: **how does MVL actually make money from [[ar-service]]?**

The honest answer requires two separate tracks.

#### Track A — 4 Things Verified in Code

| # | Name | Where it's taken | Location / evidence |
|---|---|---|---|
| ① | **Platform Fee** | A % of the **investor's profit** | `PLATFORM_FEE_PERCENT` env var, `purchase-fee.service.ts` |
| ② | **Protocol Fee** | Deducted on-chain when the investor redeems | `protocolFeeUsdc` field on the `FractionsRedeemed` event |
| ③ | **MVL Guarantor fee** | When MVL itself is a [[보증인 Guarantor]] | `isInternalMvl` flag, `GuaranteePosition.fee` |
| ④ | **MVL Seller origination** | When MVL itself is the [[판매자 Seller]], spread from originating the receivable | `isMvlInternal` flag |

##### ① Platform Fee

```
guaranteeFee   = totalProfit × (guaranteeFee% / 100)   ← guarantor's cut
platformFee    = totalProfit × (PLATFORM_FEE_PERCENT)  ← MVL's cut
net to investor = faceValue − guaranteeFee − platformFee
```

> [!warning] `PLATFORM_FEE_PERCENT` currently defaults to **0%**
> The mechanism to take the cut is wired in, but **the default is 0%**. It's a dial an operator can turn up via env var at any time, not the current main revenue source. This is the decisive clue that **MVL is not running this as a fee-extraction business**.

##### ② Protocol Fee (`protocolFeeUsdc`)

When an investor redeems at maturity, the on-chain `MusubiARProtocol` contract emits `FractionsRedeemed`, which decodes into 3 fields: `redemptionAmountUsdc / protocolFeeUsdc / guarantorFeeUsdc`. So there's **another fee taken at the on-chain layer**. [[ar-service]] simply records the value.

##### ③ MVL Guarantor fee (`isInternalMvl`)

When MVL registers itself in the [[Guarantor Registry]]. In this case the guarantee fee (`GuaranteePosition.fee`) — **a % of the investor's profit** — flows to MVL.

```
e.g. faceValue 3,000,000 KRW, purchased at 2,800,000 → profit 200,000
     fee 5% → guaranteeFee = 10,000 → MVL revenue
```

> [!note] Looking at Scenario A
> If MVL itself acts as the guarantor on [[TADA]] driver Sophea's receivable, MVL gets (a) better risk pricing because [[TADA]] already knows the driver well, (b) guarantee-fee revenue, and (c) automatic recovery through [[TADA]] earnings deduction if he defaults. **Risk control and revenue end up bundled into the same flow.**

##### ④ MVL Seller origination (`isMvlInternal`)

When MVL is registered as the [[판매자 Seller]] and originates the receivable itself. The spread between face value and what's received from [[Purchase Escrow]] = **origination revenue**.

#### Track B — Not in Code, but Implied by the Business Model

> [!example] If you only read the code, you'd ask: "Platform fee is 0%, so how does this company stay alive?"
> The answer: **this platform isn't the business, it's a booster for the real business.** The real money comes from the core.

| # | Name | How |
|---|---|---|
| ⑤ | **Net Interest Spread** | When MVL plays seller/guarantor/originator all at once, it secures a spread between cheap origination and the price investors pay |
| ⑥ | **TADA Core Synergy + Lock-in** | Sell EV bikes/cars to [[TADA]] drivers on installment → drivers locked into [[TADA]] → ride revenue ↑ + churn ↓ |
| ⑦ | **MVL Token Utility** | [[Guarantor Vault]] accepts USDC AND MVL as collateral → demand/utility for the MVL token ↑ |

##### ⑥ Synergy is the biggest piece

```
                    ┌─────────────────────────────────────┐
                    │              MVL Group               │
                    │                                      │
   Driver Sophea ─► │ [TADA app]  ────────────────┐        │
   (Payer)          │     │                       │        │
                    │     │ deducts ride earnings │        │
                    │     ▼                       │        │
                    │ [ar-service] ◄── guarantor fee       │
                    │     │                       │        │
                    │     │ tokenize receivable   ▼        │
                    │     ▼                  [Mekong Motors]│
                    │  [sell to investors] ─► cash         │
                    └──────────┬──────────────────┘
                               ▼
                       Investor (external)
```

In this picture, MVL isn't really making money from "the fee box". The real bonuses are:

1. Sophea can now afford the EV bike → [[TADA]] driver pool expands
2. Auto-deducted from Sophea's rides → he's locked to [[TADA]] → can't churn
3. Capital is supplied by external investors → MVL grows its driver fleet without burning its own balance sheet
4. Guarantee fees and platform fees stay as optional dials it can crank later

##### ⑦ MVL Token Utility

[[Guarantor Vault]] accepts **MVL tokens** alongside USDC as collateral (`SupportedTokenConfig: BASE_MVL/MVL → GuarantorVaultToken.MVL`). More guarantors → more MVL locked in vaults → contributes to token utility, demand, and price stability.

> [!warning] But MVL token value is NOT priced by [[ar-service]]
> [[ar-service]] does **not** do MVL→USD conversion itself. It only calls `vaultContract.read.totalUSDValue(bytes32 guarantorId)` on the on-chain contract, which returns the value already converted to USDC (6 decimals). Because of volatility, **real-time on-chain valuation is the only safe approach**.

---

### 3. The Real Thesis — "Wrap finance around the core, grow the ecosystem"

> [!tip] One-line summary
> MVL's revenue model is **not a fee business — it's a core-business booster**. [[ar-service]] is an **embedded-finance layer** that pulls external capital in to put productive assets under [[TADA]] drivers' wheels, while MVL optionally clips 4 kinds of fees and simultaneously grows core lock-in and token utility.

Reading the code alone, you'd wonder "if the platform fee is 0%, how does this company survive?" — but **the single word `TADA` explains everything**. Why `defaultPaymentMethod = TADA` is baked as an IPFS NFT attribute, why MVL puts itself in as guarantor, why it's fine to keep the platform fee at 0% — all because **the core business is the real engine**.

### Whole Flow Through Analogy (Baby Version + Brain Rot)

#### Baby Version — The Neighborhood Stationery Shop Tale

Once upon a time in a small village, there was a stationery shop. Follow what happens in this village and you'll understand what [[ar-service]] does, end to end.

> [!example] Meet the friends
> - 🍪 **Shop Owner** = [[seller]] (sells on credit, wants cash fast)
> - 🧒 **IOU Friend** = [[payer]] (debtor paying monthly with allowance)
> - 📜 **IOU Note** = [[invoice]] (paper saying "I'll give you 1000 won in a month")
> - 💸 **Investor Friend** = [[investor]] (buys note cheap, collects full value at maturity)
> - 💪 **Reliable Big Bro** = [[guarantor]] (promises to pay if IOU Friend doesn't)
> - 🐷 **Big Bro's Piggy Bank** = [[guarantor-vault]] (Big Bro pre-deposits coins)
> - 🐰 **Notebook Secretary Bunny** = [[ar-service]] (doesn't touch money, only organizes the notebook)
> - ✨ **Magic Safe** = [[ar-protocol-contract]] (a vending machine — money only moves by rules)

#### 1️⃣ An IOU Note Is Born — and Sliced Like a Cake into 10 Pieces

IOU Friend grabs a 1000-won snack on credit. Shop Owner gets an [[invoice|IOU note]] saying "I'll pay 1000 won in a month." But Owner can't wait a month. So they slice this note **like a cake into 10 pieces**. ([[fraction]])

```
   [ IOU Note 1000 won ]
          ↓ cake-cut
   🍰🍰🍰🍰🍰🍰🍰🍰🍰🍰
   (one slice = 100 won)
```

> [!tip] Why slice?
> - **Pocket-money friendly** — can't afford the full 1000, but you can grab a 100-won slice.
> - **Risk gets shared** — one buyer cries alone if it breaks; ten buyers each cry just a little.
> - **Each slice = a stamp (token)** — the magic safe remembers who owns which slice.

#### 2️⃣ Traffic-Light Stages — Five Faces the Note Wears

The note can't just walk onto the shelf. It changes outfits step by step. This is the [[invoice-status|status]] **traffic light**.

```
🛌 DRAFT (napping)
   ↓ "confirmed!"
🧼 CONFIRMED (washed up, dressed)
   ↓ "snap a photo to the locker, stamp the magic notebook"
🟢 LISTED (on the shelf!)
   ↓ "one or two investors buy slices"
🟡 PARTIALLY_FUNDED (half slices sold)
   ↓ "all sold!"
🔴 SOLD_OUT (every slice gone)
```

> [!note] Why traffic lights?
> At red, you can't buy. Only at green (`LISTED`) can investors purchase. Without these lights, a fake note that hadn't been stamped yet could still be sold. The lights enforce **safety rules**. The repayment lane (`UNPAID → PARTIALLY_PAID → OVERDUE → MISSED → PAID/DEFAULTED`) is the same idea on the other side.

#### 3️⃣ Internet Locker Ticket Number — IPFS

Stuffing the whole IOU note into the magic notebook (blockchain) would be too heavy. So we **photograph** the note, drop the photo into the internet locker ([[IPFS]], [[Pinata]]) and get a **ticket number** ([[ipfsCid]]) back.

```
   📜 note original
       ↓ snap!
   ☁️  internet locker (IPFS)
       ↓ ticket issued
   🎫 ticket: "Qm...abc123"
       ↓
   ✨ notebook only stores the ticket
```

> [!warning] No ticket, no listing
> Without the ticket, the magic safe refuses with "this might be a fake!" That's why the ticket **must** exist before the note can turn LISTED.

#### 4️⃣ Big Bro's Piggy Bank — The Collateral Vault

When [[guarantor|Big Bro]] swears "if IOU Friend can't pay, I'll cover 80%!", to prove he'll keep his word he pre-loads coins into the **piggy bank**.

```
   🐷 Big Bro's Piggy Bank ([[guarantor-vault]])
   ┌──────────────────────────┐
   │  💵 USDC coins (stable)   │
   │  🪙 MVL coins (volatile)  │
   └──────────────────────────┘
        ↑          ↑
        │          └── Daily Inspector
        │              (CRON_AR_SERVICE_CHECK_GUARANTOR_COLLATERAL)
        │              "All notes maturing in next 7 days —
        │               is the piggy bank that full?"
        │
        └── Burglar camera
            (MonitorVaultMovementUc)
            "stranger wallet touches it → Slack alert!"
```

> [!tip] Who knows the MVL coin's worth?
> Bunny can't price MVL by itself. It asks the magic safe directly — "what's this piggy bank worth in USD right now?" ([[totalUSDValue]]). Coin prices wobble, so a live answer is the safest answer.

#### 5️⃣ Pricing Strategies and the Mailbox

Owner picks ahead of time how to discount the note. There are three **melting ice-cream** strategies.

| Strategy | Analogy | Behavior |
|----------|---------|----------|
| `FIXED_APR` | Fixed price 🏷️ | Same % whenever you buy |
| `LINEAR_DECAY` | Slide 🛝 | APR drops in a straight line over time |
| `STEP_DECAY` | Stairs 🪜 | APR drops in jumps weekly/monthly |

And when Bunny goes to stamp the magic notebook, the road is sometimes blocked. So we write the chore into a **mailbox** ([[outbox]], [[OnchainTransaction]]) and Bunny retries **up to 5 times**.

```
   📬 mailbox
   ┌─────────────────────────────┐
   │ [PENDING] stamp the safe     │
   │ [PROCESSING] claimed, trying │
   │ [COMPLETED] success!         │
   │ [FAILED → back to PENDING]   │
   │ [PERMANENTLY_FAILED — 5 fails]│
   │  └─ Slack: call a grown-up 📢│
   └─────────────────────────────┘
```

#### 6️⃣ Daily Inspection — Can Big Bro Keep His Word?

Every day Inspector (a cron job) wakes up and asks:

> "Add up all notes maturing in the next 7 days. Is 80% of that sitting in Big Bro's piggy bank?"

If short, off goes an email to Big Bro: "Please top up coins!"

#### 7️⃣ Happy Ending vs Sad Ending

```
   ┌──── If IOU Friend pays (happy 😊) ────┐
   │                                        │
   IOU Friend → 💵 Payback Escrow (repayment safe)
              ↓ operator records payment event
              ↓ Bunny syncs to the magic notebook
              ↓ at maturity, redeem window opens
   Investor  ← 💵 (recovered with interest)
   
   
   ┌──── If IOU Friend defaults (sad 😢) ──┐
   │                                        │
   IOU Friend → ❌ (no money sent)
              ↓ DEFAULTED verdict
              ↓ magic safe triggers automatically
   🐷 break open Big Bro's piggy bank!
              ↓ up to coverageRatio (80%)
   Investor  ← 💵 (mostly recovered, small loss)
```

> [!note] Bunny never touches money
> Real money moves through the **Magic Safe Trio**:
> - [[purchase-escrow]] = pools investor cash, hands it to the shop owner
> - [[payback-escrow]] = pools IOU Friend's repayments, hands them to investors
> - [[guarantor-vault]] = holds Big Bro's collateral
>
> Bunny ([[ar-service]]) only tends the notebook and asks the safe for stamps.

#### 8️⃣ One-Page Diagram

```
                  🐰 Bunny the Secretary (ar-service)
                  "I just organize the notebook"
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
   📜 mint notes   🎫 get ticket    📬 ask via mailbox
   (Contract→        (IPFS)           for stamps
    Invoice)                          (Outbox→Onchain)
                       │
                       ▼
              ✨ Magic Safe (Smart Contract)
              "vending machine — code is law"
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       ▼               ▼               ▼
   💵 Purchase     💵 Payback       🐷 Guarantor
      Escrow          Escrow            Vault
   (investor $)    (repayment $)     (collateral)
```

---

### Brain Rot Edition (optional chapter, alarms off)

> [!warning] 0% seriousness
> Just for laughs. The baby version above is the source of truth.

Once upon a time there was a snack shop ngl, Shop Owner ([[seller]], cash-strapped sigma grindset L bozo), IOU Andy ([[payer]], "I'll pay when allowance drops fr"), and the IOU note got reborn as a ✨NFT✨ aka [[invoice]]. Cake-cut into 10 slices ([[fraction]]) — investor brOs ([[investor]]) sliding in for a slice each. Buy at 950, get 1000 → +50 ez money [[APR]] mooning no cap fr.

But what if IOU Andy ghosts? Two paths fam:
- **No-guard / unguaranteed** ([[isGuaranteed|isGuaranteed=false]]): you eat the loss yourself, L take.
- **Sigma Big Bro guaranteed** ([[guarantor]]): Big Bro pre-stashes USDC/MVL in the piggy bank ([[guarantor-vault]]) saying "I gotchu." Charges fanum tax for the favor, and if Andy bounces, Bro's piggy gets cracked open.

Piggy gets inspected every day (`CRON_AR_SERVICE_CHECK_GUARANTOR_COLLATERAL`) — if the next 7 days of obligations aren't 80% covered, email shoots out: "top up coins bro." MVL price is asked straight from the magic safe ([[totalUSDValue]]) because coin prices wobble like crazy ong.

There's a thief watch too ([[MonitorVaultMovementUc]]) — random wallet sniffing the piggy? among us SUS 🚨 Slack `@channel` ping ddd. Imposter ejected on the spot.

Pricing strats trio:
- `FIXED_APR` — fixed-price (no cap no rizz)
- `LINEAR_DECAY` — slide (APR keeps dropping)
- `STEP_DECAY` — staircase (APR plummets weekly)

Magic-safe stamp blocked? [[outbox|Mailbox]] queues the chore and retries up to 5 times — five strikes and it's `PERMANENTLY_FAILED`, Slack ping incoming. Money work going silent? That's NPC behavior, not allowed.

If Andy pays? Payback Escrow stacks up → investor brOs hit redeem at maturity → 💰 cashed out. **If Andy ghosts?** Magic safe cracks open Bro's piggy (fanum tax for real this time) → covers up to coverageRatio (80%). Unguaranteed? Straight-up L take.

Key truths fr:
- [[ar-service]] = secretary bunny (no money touch, just vibes)
- [[ar-protocol-contract|magic safe]] = the actual wizard (code is law, vending-machine moves)
- [[TADA]] = MVL's day job (Cambodia ride-hailing superapp, drivers' fares auto-deducted = `defaultPaymentMethod=TADA`)

Full flow in one line: **Shop Owner needs cash → IOU note becomes an NFT → sliced into pieces → Big Bro stacks collateral → investor brOs scoop slices → cash out at maturity ✨ or crack open Bro's piggy on default**. That's the RWA invoice game no cap fr fr 🗿.

## Diagram

[[canvas/AR-Service_기반_RWA_인보이스_투자_상품_전체_구조와_플로우.canvas|Concept map]]
