---
id: 019ef769-996f-75ec-8f2b-7862af4ecb4d
title: ar-service 인보이스 투자 상품 플로우
topics:
  - ar-service
  - invoice
  - redeem
  - escrow
  - guarantor
  - marketplace
tags:
  - ar-service
  - RWA
  - redeem
  - escrow
  - investor
  - USDC
  - 교육용
  - 한국어
  - continuation
sources:
  - 019ef370-2e84-701d-b3b9-7f3dab509e36
created_at: '2026-06-24T02:15:53.455Z'
updated_at: '2026-06-24T02:15:53.455Z'
---
## TL;DR

`ar-service` 인보이스 투자 상품은 **실물 차량이 아니라 매출채권(invoice/받을 권리)** 을 NFT 조각으로 토큰화해 판매하는 구조다. 컨트랙트 이름이 `MusubiARProtocol` / `MusubiARNFT`라 [[musubi]] 브랜드를 공유하지만, IPFS 메타데이터에 담기는 건 `faceValue` · `maturityDate` · `seller` · `payer` · `contractId` 뿐이고 VIN/주행거리 같은 차량 식별자는 0건이다. 투자자 측 라이프사이클은 **100% 온체인 [[USDC]]** 로 굳어져 있어 — 구매도, 만기 redeem도 USDC, redeem 엔드포인트는 `txHash` 필수 — 채무자만 현금 상환이 가능한 **비대칭 통화 설계**다. 또 한 가지 핵심은 **시간차 단계**다: 운영자가 [[Seller]] · [[Payer]] (+보증이면 [[Guarantor]])를 먼저 등록·`ACTIVE`로 만들고 `generateInvoices`로 쪽지를 생성해 `LISTED`까지 올린 뒤에야 마켓이 열리며, [[Investor]]는 그 후에 등록(`POST /ar/investor`)하고 구매한다 — "4명 다 모아놓고 시작"이 아니라 채권 발행 측과 투자 측이 **순차적으로 합류**한다.

> [!note] 한 줄 멘탈 모델
> **무엇을 토큰화하나?** 차량이 아니라 매출채권. **누구의 돈으로?** 투자자는 무조건 USDC. **언제?** 발행 측 4-게이트 통과 → `LISTED` → 마켓 오픈 → 투자자 등록 후 구매. 이 세 축이 흔들리지 않는 골격.

## 바인딩 대상: 차량이 아니라 매출채권

ar-service의 투자 상품은 **실물 차량과 묶여 있지 않다.** 토큰화되는 대상은 "차를 팔고 받을 돈" — 즉 [[매출채권(Account Receivable)]] 그 자체다. 코드 베이스 전체에서 `vehicle`, `VIN`, `asset`, `odometer` 같은 키워드를 grep하면 **히트 0건**이고, IPFS에 올라가는 인보이스 메타데이터에도 차량 식별자는 한 글자도 들어가지 않는다.

> [!note] 핵심 정리
> ar-service는 **채권(받을 권리)** 을 토큰화한다. [[Musubi]]가 차량 자체를 토큰으로 묶는 모델이라면, ar-service는 "누가, 누구에게, 얼마를, 언제까지 갚느냐"만 추적한다. 차량은 기껏해야 그 채권이 생겨난 오프체인 배경일 뿐, 시스템의 시야 안에는 들어오지 않는다.

### IPFS 메타데이터에 들어가는 것 (그리고 들어가지 않는 것)

`invoice-ipfs.service.ts:80` 부근에서 메타데이터를 조립할 때 실제로 직렬화되는 필드는 다음 5개뿐이다:

| 필드 | 의미 | 차량 관련? |
|---|---|---|
| `faceValue` | 채권 액면가 | ✗ |
| `maturityDate` | 만기일 | ✗ |
| `seller` | 채권 매도인 (원 채권자) | ✗ |
| `payer` | 채무자 (만기에 돈을 갚을 주체) | ✗ |
| `contractId` | 원 계약 식별자 | ✗ |

VIN, 차대번호, 차종, 주행거리, 차량 ID, 어떤 종류의 자산 식별자도 없다. 즉 NFT 한 장이 가리키는 건 "특정 차"가 아니라 "특정 채권"이다.

```
[ 실물 차량 ]   ←  ar-service의 시야 밖
     │
     │ (오프체인 매매 → 외상 발생)
     ▼
[ 매출채권 ]   ←  토큰화 대상
     │
     │  faceValue / maturityDate / seller / payer / contractId
     ▼
[ IPFS 메타데이터 ]  →  [[MusubiARNFT]] mint
```

### Musubi와의 차이 — 이름은 공유, 바인딩은 다름

컨트랙트 이름이 `MusubiARProtocol`, `MusubiARNFT`로 [[Musubi]] 브랜드를 공유하기 때문에 헷갈리기 쉽지만, **바인딩 계층이 완전히 다르다.**

| 구분 | [[Musubi]] (추정) | [[ar-service]] |
|---|---|---|
| 토큰화 대상 | 차량 자체 | 매출채권 (AR) |
| 식별자 | VIN / 차량 ID | contractId / payer |
| 만기 개념 | (없음, 보유 자산) | maturityDate 있음 |
| 상환 트리거 | 차량 매각/회수 | 채무자 변제 |
| 디폴트 의미 | 차량 멸실/탈취 | 채무자 미상환 |

이름만 닮았지, ar-service의 NFT는 "이 차의 소유권 조각"이 아니라 **"이 매출채권의 분할 지분"** 이다.

### vehicleOracle / musubiService — 선언만 있고 호출은 0건

`infra.config`에는 `vehicleOracleService`, `musubiService`의 HTTP 슬롯이 선언돼 있다. 처음 본 사람은 "아, 차량 시세나 상태를 외부에서 가져오는구나"라고 오해하기 쉽다. 그런데 실제 호출 지점을 grep해 보면:

```bash
$ grep -rn "vehicleOracleService\." src/
# (선언부 외 호출 0건)

$ grep -rn "musubiService\." src/
# (선언부 외 호출 0건)
```

**투자 상품 라이프사이클 어디에서도 호출되지 않는다.** 인보이스 생성, 마켓 오픈, 구매, 만기 redeem, 디폴트 처리 — 어느 경로에도 차량 오라클을 찌르는 로직이 없다.

> [!warning] 함정: 설정 슬롯 ≠ 실제 의존성
> `vehicleOracleService` 슬롯이 환경 변수에 있다고 해서 "차량 가격에 따라 무언가가 움직인다"고 추론하면 안 된다. 실제 동작에는 영향이 없는 **레거시/플레이스홀더**일 가능성이 높다. 향후 차량 평가 연동을 염두에 둔 자리만 잡혀 있는 셈이다.

### 그래서 이게 왜 중요한가

채권 바인딩 모델이라는 사실은 다른 모든 섹션의 전제를 바꾼다:

- **만기 트리거**가 차량 매각이 아니라 `payer`의 변제이므로, [[통화/에스크로/만기 수령 구조]]가 "Payer가 갚은 USDC → Payback Escrow → 투자자 redeem"으로 짜인다.
- **디폴트 시나리오**가 차량 멸실이 아니라 채무자 미상환이므로, [[보증 수수료와 디폴트/부분(partial) 시나리오]]에서 [[Guarantor Vault]]가 등장한다.
- **참여자 등록**에서 등록 대상이 차량 보유자가 아니라 `seller`/`payer`/(옵션) `guarantor`인 이유도 여기서 나온다 — 시스템은 "이 차를 누가 가지고 있느냐"가 아니라 "이 채권을 누가 발행하고 누가 갚느냐"만 본다.

> [!example] 한 줄 요약
> ar-service는 **차가 아니라 빚을 토큰으로 쪼갠다.** IPFS 메타데이터에는 `faceValue/maturityDate/seller/payer/contractId`만 들어가고, `vehicleOracle`·`musubiService`는 선언만 있을 뿐 호출되지 않는다.

## 참여자 등록과 '시작'의 의미

이 섹션의 핵심은 두 가지다. ① **상품을 만들기 전에 등록이 필요한 참여자가 누구인지** ② **"시작한다"가 코드 레벨에서 정확히 어떤 호출을 의미하는지**. 직관적으로는 "Buyer / Seller / Payer / Guarantor 4명을 다 등록하고 시작한다"고 말하기 쉬운데, 실제 [[ar-service]] 구현은 그렇지 않다.

### 등록 = DB ACTIVE 즉시 + 온체인 Registry 비동기

먼저 "등록"이라는 한 단어가 실제로 두 개의 작업을 트리거한다는 점을 분리하자.

| 단계 | 위치 | 타이밍 | 상태 필드 |
|---|---|---|---|
| 1) 도메인 레코드 생성 | Prisma DB | 동기 (즉시) | `status = ACTIVE` (`@default(ACTIVE)`) |
| 2) 온체인 Registry 등록 | [[Outbox]] → SC | 비동기 | `onchainStatus: PENDING → CONFIRMED` |

```
register(participant)
  ├─ INSERT participant (status=ACTIVE)              ── 동기, 트랜잭션 내
  └─ INSERT outbox (op=REGISTER, onchainStatus=PENDING)
        ↓ (워커가 픽업)
        sendTx(Registry.register(...))
        ↓ (이벤트 수신)
        UPDATE participant SET onchainStatus=CONFIRMED
```

> [!note] DB ACTIVE는 즉시, 온체인은 한 박자 늦는다
> Seller/Payer/Guarantor를 등록하면 DB에서는 곧바로 `ACTIVE`가 되지만, 온체인 Registry에 반영되는 데에는 outbox 워커 사이클이 필요하다. 다음에 설명할 `generateInvoices` 게이트가 **DB ACTIVE만 확인**하는 것은 바로 이 비동기 갭 때문이다.

### "등록이 필요한 참여자"는 사실 3명뿐 (Buyer 제외)

`generate-invoices.usecase.ts`를 뒤져 보면 `investor` / `buyer`를 참조하는 코드가 **0건**이다. 즉, 상품(인보이스 토큰) 생성 단계의 전제는 다음 셋뿐:

- [[Seller]] — 매출채권을 파는 쪽 (자금이 필요한 기업)
- [[Payer]] — 채권을 갚을 채무자
- [[Guarantor]] — *옵션*. 보증이 붙는 상품일 때만, 그것도 `GuaranteePosition` 등록 단계의 전제이지 인보이스 생성 자체의 전제는 아님.

[[Investor]] (buyer)는 전제가 아니다. 마켓이 오픈된 뒤 본인이 `POST /ar/investor`로 직접 등록한다 (자세한 시점은 [[마켓 오픈과 투자자 합류 시점]] 섹션 참고).

> [!warning] "4명 다 등록하고 시작" 표현이 만드는 오해
> 운영 관점에서 buyer는 *그 다음 단계*의 주체다. 상품을 만들 때는 매출채권의 양 당사자 (Seller↔Payer) + 보증 (Guarantor) 만으로 충분하다. Investor는 마켓에 인보이스가 노출된 이후 "스스로 가입하고 들어오는" 흐름이다.

### "시작한다" = `generateInvoices` 호출

"시작한다"의 정확한 의미는 **`generateInvoices` 유스케이스를 호출해 Contract 한 건에 대응하는 인보이스(쪽지) 레코드들을 생성하는 것**이다. 이 호출이 통과해야 인보이스가 `DRAFT` 또는 그 이후 상태로 존재하게 되고, 이후의 모든 흐름(마켓 리스팅, 구매, 만기 redeem)이 가능해진다.

### 4개의 게이트

`generateInvoices` 진입 시 다음 네 가지가 모두 충족돼야 한다:

```
                 generateInvoices(contractId)
                          │
        ┌─────────────────┼─────────────────┬─────────────────┐
        ▼                 ▼                 ▼                 ▼
 ① Contract.status   ② Contract.payer   ③ Seller.status   ④ Payer.status
    === DRAFT          Confirmation         === ACTIVE        === ACTIVE
                       === RECEIVED         (DB)              (DB)
```

| # | 게이트 | 의미 |
|---|---|---|
| ① | `Contract.status = DRAFT` | 아직 인보이스가 발행되지 않은 계약만 시작 대상 |
| ② | `Contract.payerConfirmation = RECEIVED` | Payer가 "이 채무 인정한다"를 회신한 상태 |
| ③ | `Seller.status = ACTIVE` (DB) | Seller가 등록되어 운영 가능 상태 |
| ④ | `Payer.status = ACTIVE` (DB) | Payer가 등록되어 운영 가능 상태 |

> [!tip] 게이트는 **DB ACTIVE**만 본다 — `onchainStatus`는 보지 않는다
> `generateInvoices`는 Seller/Payer의 `onchainStatus === CONFIRMED`를 *확인하지 않는다*. 그 결과 온체인 Registry 등록이 아직 `PENDING`인 상태에서도 인보이스 `DRAFT`를 만들 수 있다.
>
> **그래도 괜찮은 이유**: 인보이스를 마켓에 올리거나(SC 등록 단계) 투자자가 구매하려면 그때는 인보이스 자체의 `onchainStatus = CONFIRMED`가 요구된다(`PURCHASABLE_INVOICE_STATUSES = [LISTED, PARTIALLY_FUNDED]` + `onchainStatus = CONFIRMED`). 이 시점이 되기까지 Seller/Payer 등록 outbox가 이미 confirm 됐을 가능성이 높다.
>
> **여전히 주의할 점**: 만약 DB ACTIVE인데 온체인 Registry는 아직 PENDING인 채로 인보이스 SC 등록을 진행하면 revert가 날 수 있다. 운영적으로 outbox confirm을 기다리는 게 안전하다.

### Investor 등록은 게이트 모양이 다르다

비교를 위해 [[Investor]] 쪽 등록도 짚어두면:

- Prisma 모델에 `@default(ACTIVE)` — **승인 절차 없음**, 등록 즉시 `ACTIVE`
- 온체인 Registry 등록 **없음** (outbox 발생 X)
- `userId` 멱등 체크 → [[user-service]]에서 메인 지갑 조회(없으면 404) → 지갑 중복 체크(409) → email/name 채워 생성
- 메인 지갑 필수

즉, 같은 "ACTIVE"라도 Seller/Payer/Guarantor의 ACTIVE는 *온체인 Registry까지 가는 비동기 꼬리*가 있고, Investor의 ACTIVE는 *DB 한 방으로 끝*이다.

> [!example] 운영 시나리오 한 줄 정리
> 1) Seller 등록 → DB ACTIVE 즉시, 온체인 PENDING → CONFIRMED (비동기)
> 2) Payer 등록 → 동일
> 3) (옵션) Guarantor 등록 → 동일. 단 보증 상품에서 `GuaranteePosition`을 붙일 때 전제
> 4) Contract `DRAFT` 생성 + Payer가 회신해 `payerConfirmation = RECEIVED`
> 5) **여기서 "시작" = `generateInvoices(contractId)` 호출** → 게이트 4개 통과 시 인보이스 DRAFT 생성
> 6) 인보이스 SC 등록 → `onchainStatus = CONFIRMED` → 마켓 LISTED
> 7) **이때부터** Investor들이 와서 등록하고 구매

## 마켓 오픈과 투자자 합류 시점

앞 섹션에서 "Seller/Payer/(Guarantor)를 먼저 등록하고 상품을 만든다"고 했을 때 자연스럽게 드는 의문이 "그럼 투자자(buyer)도 미리 등록돼 있어야 하나?"입니다. 결론부터 말하면 **아니다** — 마켓은 상품이 만들어진 뒤에 열리고, 투자자는 그 뒤에 자기 발로 등록해서 들어옵니다.

### 시간 축으로 본 두 단계

```
[T0] 운영자 측 작업                     [T1] 마켓 오픈            [T2] 투자자 합류
─────────────────────────              ─────────────────         ─────────────────
- Seller 등록 (ACTIVE + 온체인)         - invoice 상태 = LISTED   - GET /ar/market/listing
- Payer  등록 (ACTIVE + 온체인)         - GET /ar/market/listing  (구경, 인증 불필요)
- (Guarantor 등록 + GuaranteePosition)    가 Public 으로 노출      ─────────────────
- Contract DRAFT 생성                                              - POST /ar/investor
- generateInvoices() 호출                                            (본인 등록, JWT 발급)
- invoice → LISTED + saleStartDate                                 - POST /ar/purchase
                                                                     (JWT AR_INVESTOR)
```

즉 "4자 등록 → 시작"이라는 표현은 약간 오해를 부르는 단축인데, 정확히는 **공급 측 3자(Seller, Payer, +선택적 Guarantor)** 만 상품 생성의 전제이고, **수요 측(Investor)** 은 시간차를 두고 마켓에 합류합니다.

### 인증 경계가 그대로 코드에 박혀 있음

마켓 컨트롤러의 권한 데코레이터를 보면 이 두 단계가 그대로 드러납니다.

| 엔드포인트 | 인증 | 용도 |
|---|---|---|
| `GET /ar/market/listing` | **Public** | 비로그인 누구나 구경 — 주석에 *"investor can connect wallet later"* |
| `GET /ar/market/listing/:id` | **Public** | 단일 인보이스 상세 |
| `POST /ar/investor` | Public (user JWT) | 본인을 [[ar-service-investor]]로 등록 |
| `POST /ar/purchase` | **JWT AR_INVESTOR** | 등록된 투자자만 |
| `POST /ar/purchase/redeem` | **JWT AR_INVESTOR** | 등록된 투자자만 |

`get-listings.usecase.ts:75`에서 마켓에 노출되는 상태는 `LISTED`, `PARTIALLY_FUNDED`, `SOLD_OUT` 세 가지로 좁혀집니다 — DRAFT나 REDEEMED는 마켓에 뜨지 않습니다.

> [!note] 마켓의 비대칭 접근
> **구경은 누구나, 구매는 등록된 투자자만.** 지갑 연결 전에 상품을 둘러보고 마음에 들면 그때 [[POST /ar/investor]]로 등록하는 동선이 의도된 설계입니다.

### 투자자 등록 = 즉시 ACTIVE

`create-investor.usecase`의 흐름은 의외로 간단합니다.

```
POST /ar/investor (user JWT)
   │
   ├─ userId 멱등 체크 (이미 등록돼 있으면 그대로 반환)
   ├─ user-service에서 메인 지갑 조회   ─→ 없으면 404
   ├─ 지갑 중복 체크                    ─→ 다른 investor가 쓰고 있으면 409
   ├─ email / name 가져와서 row 생성
   └─ INSERT Investor { status: ACTIVE }   ← Prisma @default(ACTIVE)
```

핵심 특징:

- **승인 절차 없음** — `PENDING → APPROVED` 같은 워크플로우 자체가 없음. Prisma 스키마에서 `status` 기본값이 그대로 `ACTIVE`.
- **메인 지갑이 필수** — [[user-service]]에 메인 지갑이 등록돼 있지 않으면 404로 막힘.
- **온체인 Registry 등록이 없다** — Seller/Payer/Guarantor와 달리 투자자는 outbox에 `REGISTER` 이벤트를 쏘지 않음. 즉 [[onchainStatus]] 필드 자체가 존재하지 않고, 컨트랙트는 투자자가 누군지 모름. 컨트랙트 입장에서 투자자는 그냥 "쪽지를 들고 온 어떤 지갑 주소"일 뿐입니다.

> [!tip] 왜 투자자만 온체인 등록이 없나
> Seller/Payer/Guarantor는 **인보이스에 박혀 검증되는 주체**(채권 발행자, 채무자, 보증인)지만, 투자자는 **NFT 보유자**일 뿐입니다. ERC-721/1155가 이미 "지갑이 토큰을 갖고 있는지"를 검증해 주므로 별도 화이트리스트가 필요 없습니다.

### 구매 게이트 5 + 1

등록만 됐다고 아무 인보이스나 살 수 있는 게 아니라, `purchase-investor.usecase.ts`가 매 구매 요청마다 6개 조건을 통과시킵니다.

```ts
const PURCHASABLE_INVOICE_STATUSES = [
  InvoiceStatus.LISTED,
  InvoiceStatus.PARTIALLY_FUNDED,
]; // :37
```

| # | 검사 | 라인 | 실패 시 |
|---|---|---|---|
| 1 | `invoice.status ∈ {LISTED, PARTIALLY_FUNDED}` | `:173` | 400 — DRAFT/SOLD_OUT/REDEEMED 차단 |
| 2 | `invoice.onchainStatus === CONFIRMED` | `:179` | 400 — DB는 LISTED여도 SC 발행 대기 중이면 막힘 |
| 3 | `now ≥ saleStartDate` | `:186` | 400 — 판매 시작 전 |
| 4 | `now ≤ saleEndDate` | `:189` | 400 — 컷오프 이후 (`saleEndDate = dueDate − cutoffDays`) |
| 5 | `invoice.availableCount ≥ 요청 조각수` | `:193` | 400 — 남은 조각보다 많이 사려 함 |
| +1 | `investor.status === ACTIVE` | `:67` | 403 — 등록 안 됨/비활성 |

> [!example] 두 개의 "active"가 동시에 맞아야 거래 성사
> - **투자자 active**: `Investor.status === ACTIVE` (등록 즉시 충족)
> - **인보이스 구매 가능 상태**: 위의 1~5번 조건 묶음
>
> 어느 한쪽이라도 빠지면 구매 실패. 마켓에 떠 있다고 무조건 살 수 있는 게 아니라, 시간 창(`saleStart ~ saleEnd`)과 잔여 조각이 같이 살아 있어야 한다.

### saleStartDate / saleEndDate가 만드는 시간 창

`generateInvoices` 단계에서 `cutoffDays`를 빼서 `saleEndDate`가 계산됩니다.

```
saleStartDate ───────── 판매 가능 구간 ───────── saleEndDate ───── dueDate(만기)
                                                  │  ←cutoffDays→  │
                                                  │
                                          여기를 넘기면 신규 매수 불가,
                                          기존 보유분만 redeem 대기
```

만기 직전까지 사람을 받으면 redeem 정산이 꼬이므로, 컷오프 일수만큼 미리 마켓을 닫아 두는 안전 마진입니다. 마켓 상태가 `LISTED`로 남아 있어도 4번 게이트(`now ≤ saleEndDate`)에서 차단되므로, "보이는 것"과 "살 수 있는 것"이 시간 축에서 또 한 번 갈라집니다.

### 종합 — 합류는 한 번에 끝나는 절차

따라서 투자자의 합류 시퀀스는 다음 한 줄로 요약됩니다.

```
지갑 연결 → POST /ar/investor → 즉시 ACTIVE → JWT 발급 → POST /ar/purchase
   (user JWT)                    (DB only,         (AR_INVESTOR)    (게이트 5+1 통과 시
                                  온체인 X)                          PurchaseTransaction 생성)
```

운영자가 관여하는 승인 큐도, KYC 대기열도, 온체인 화이트리스트 트랜잭션도 없습니다. 그만큼 "투자자 합류"는 마찰 없이 즉시 일어나도록 설계돼 있고, 진짜 검증은 **구매 시점의 6게이트**에 모여 있습니다.

## 통화/에스크로/만기 수령 구조

ar-service는 통화(Currency)를 **세 가지 축으로 분리**해서 다루지만, 실제 자금 이동은 **단일 온체인 USDC 에스크로**로 수렴하는 비대칭 설계다. 채무자만 현금을 만질 수 있고, 투자자는 처음부터 끝까지 토큰만 만진다.

### 1) 통화 3분리 — `currencyOriginal` / `listingCurrency` / `redemptionCurrency`

[[invoice]] 한 장에는 통화 필드가 셋이다. 이름이 비슷해 보이지만 역할이 완전히 다르다.

| 필드 | 의미 | 누가 본다 | 실제 사용처 |
|---|---|---|---|
| `currencyOriginal` | 원래 채무의 표시 통화 (예: USD, USDC, JPYC) | 회계/오프체인 기록 | 채권 메타데이터, IPFS |
| `listingCurrency` | 마켓플레이스에 노출되고 투자자가 거래하는 통화 | 투자자 | **온체인 토큰 주소 매핑**의 키 |
| `redemptionCurrency` | 만기 수령 시 표시되는 통화 | 보고서 상 | redeem 로직선에서는 **실질적으로 미사용** |

> [!note] 3개 필드가 따로 존재하는 이유
> 채무는 USD로 났는데 마켓에서는 USDC로 팔고 싶을 수 있다. 회계상 표시 통화(`currencyOriginal`)와 토큰화된 거래 통화(`listingCurrency`)를 분리해 두면, 동일한 원장 데이터를 다양한 형태로 노출할 수 있다. 다만 `redemptionCurrency`는 만기 수령 로직에서 실질 게이트로 쓰이지 않는다 — 실제 지급 토큰은 `listingCurrency` 매핑이 결정한다.

### 2) USD → USDC 강제 매핑

가장 중요한 트릭은 **USD가 들어오면 무조건 USDC 토큰 주소로 치환**된다는 점이다. `invoice-onchain.service.ts:162` 부근의 매핑 로직은 대략 다음과 같다.

```ts
switch (listingCurrency) {
  case 'USDC':
  case 'USD':         // ← 현금 표시 USD도 USDC 컨트랙트로 강제 매핑
    return USDC_TOKEN_ADDRESS;
  case 'JPYC':
    return JPYC_TOKEN_ADDRESS;
  default:
    throw new UnsupportedCurrencyError();
}
```

> [!warning] '현금 USD invoice'라는 표현은 **채무자가 현금으로 갚는다**는 뜻이지, **투자자가 현금을 받는다**는 뜻이 아니다.
> 동일한 invoice에 대해 채무자는 은행 송금으로 USD를 입금할 수 있지만, 그 순간 운영자가 같은 금액의 USDC를 [[Payback-Escrow]]에 충당해야 투자자에게 redeem이 가능해진다.

### 3) 에스크로 3종 — 통화 무관, 전부 온체인 USDC

ar-service의 에스크로는 **'현금 전용 에스크로'가 존재하지 않는다.** `listingCurrency`가 무엇이든 동일한 온체인 컨트랙트 3개로 흐른다.

```
┌──────────────────────────────────────────────────────────────┐
│                    ON-CHAIN (USDC 기준)                       │
│                                                              │
│   [Purchase Escrow] ←──── 투자자 USDC 입금 (구매 시)          │
│         │                                                    │
│         │ generateInvoices/풀딩 완료 후                       │
│         ▼                                                    │
│      Seller 지갑 (선지급금 수령)                              │
│                                                              │
│   [Payback Escrow] ←──── 만기 직전, Payer 상환분 USDC 적치    │
│         │                                                    │
│         │ redeem 시                                          │
│         ▼                                                    │
│      Investor 지갑 (USDC 수령)                                │
│                                                              │
│   [Guarantor Vault] ←──── Guarantor 담보금 USDC 예치          │
│         │                                                    │
│         │ 디폴트 발생 시                                      │
│         ▼                                                    │
│      Payback Escrow / Investor 지갑 (보증 보상)               │
└──────────────────────────────────────────────────────────────┘
```

| 에스크로 | 토큰 | 출금 트리거 | 디폴트 시 행동 |
|---|---|---|---|
| **Purchase Escrow** | USDC | 펀딩 완료/판매 종료 | Seller에게 정산 |
| **Payback Escrow** | USDC | 투자자 `redeem` | 잔액 부족분은 Vault에서 충당 |
| **Guarantor Vault** | USDC | 디폴트 + 보증 발동 | Payback Escrow로 자금 이동 |

> [!tip] 통화 무관 == 컨트랙트 무관이 아니다
> 동일 invoice의 통화별로 별도 컨트랙트를 띄우는 게 아니라, `listingCurrency` → 토큰 주소 매핑만 바뀌고 에스크로 컨트랙트 자체는 동일하다. JPYC invoice라면 같은 Escrow가 JPYC 토큰을 들고 있는 식.

### 4) 채무자(Payer) 현금 상환 → 운영자 USDC 충당 패턴

채무자가 진짜 USD/원화/엔화 **현금**으로 갚는 경우, 시스템은 다음 비대칭 흐름을 탄다.

```
   Payer (채무자)                          Operator
   ─────────────                          ──────────
   1) 은행 송금 (실물 USD/JPY/KRW)   ───▶  실물 자금 수령
                                            │
                                            ▼
                                       2) `paybackFiat` 기록 (DB)
                                            │
                                            ▼
                                       3) 운영자 보유 USDC를
                                          Payback Escrow로 전송
                                            │
                                            ▼
                              ┌──── Payback Escrow (USDC 적재)
                              │
                              ▼
                        4) Investor가 redeem 호출
                              │
                              ▼
                        Investor 지갑 (USDC 수령)
```

> [!example] 핵심: **운영자가 환전 리스크와 충당 부담을 짊어진다**
> 채무자가 현금 100,000 USD를 보냈다면, 운영자는 본인이 보유한 USDC로 100,000을 Payback Escrow에 채워 넣어야 한다. 채무자→투자자 사이의 통화 간극을 운영자가 자기 자본/USDC 잔고로 메우는 구조.

### 5) 만기 수령(redeem) — `FractionsRedeemed` 이벤트 기반 USDC 지급

투자자가 만기에 자금을 받는 경로는 **단 하나**다. 진입점은 `POST /ar/purchase/redeem` (purchase.controller.ts:51).

```
  Investor 지갑                                On-chain Contract
  ─────────────                                ──────────────────
  redeem() 트랜잭션 서명/전송  ────────────▶  redeem 실행
                                                  │
                                                  │ 정상 상환: Payback Escrow에서 인출
                                                  │ 디폴트+보증: Guarantor Vault에서 인출
                                                  ▼
                                              USDC 전송 → Investor 지갑
                                                  │
                                                  ▼
                                              `FractionsRedeemed` 이벤트 emit
                                                  │
                                                  ▼
  ar-service: 이벤트 수신                ◀───── (txHash 필수, redeem-investor.dto.ts:11)
       │
       ▼
  PurchaseTransaction.redemptionAmount 기록
  guarantorFeeUsdc 차감 후 투자자 손익 계산
```

> [!note] `FractionsRedeemed` 이벤트의 결정성
> ar-service는 만기 지급액을 자체 계산하지 않는다. 온체인에서 발생한 `FractionsRedeemed` 이벤트의 페이로드(예: `redemptionAmount`, `guarantorFeeUsdc`)를 **소스 오브 트루스**로 받아 그대로 기록한다. 즉 redeem의 dto는 `txHash`가 `@IsNotEmpty`로 강제되어 있어 — **온체인 거래 없이는 redeem 자체가 불가능**하다.

### 6) 비대칭의 정리

| 역할 | 통화 입출금 형태 | 비고 |
|---|---|---|
| **Seller** | 온체인 USDC (Purchase Escrow → Seller 지갑) | 선지급 |
| **Payer (채무자)** | **현금 OK** (USD/JPY/KRW) 또는 USDC | 유일하게 현금 허용 |
| **Investor (buyer)** | **USDC 전용** | 구매도 redeem도 토큰 |
| **Guarantor** | 온체인 USDC (Vault 예치) | 담보, 보상 모두 토큰 |

> [!warning] 투자자 100% 현금 수령은 **현재 구조에서 불가능**
> redeem 진입점이 txHash를 강제하고, 수령 금액이 온체인 이벤트에서 오며, Investor 모델에 은행계좌/IBAN 필드가 없고, payout/disburse/bankTransfer 같은 오프체인 송금 레일이 코드에 0건이다. 이 주제는 별도 섹션([[투자자 100% 현금 수령은 가능한가]])에서 다룬다.

## 보증 수수료와 디폴트/부분(partial) 시나리오

이 섹션은 [[ar-service]]에서 **보증인(Guarantor)** 의 수수료가 언제 청구되는지, 여러 투자자가 한 [[invoice]]를 쪼개 산 경우 수수료가 어떻게 갈리는지, 그리고 "부분(partial)" 시나리오 4종이 각각 어디서 발생하는지를 코드 기준으로 정리한다.

### 1. 보증 수수료의 본질: "디폴트 여부"가 아니라 "투자자 이익 여부"

> [!warning] 가장 흔한 오해
> "디폴트가 났으니까 보증인이 보상해줬고, 그러면 수수료를 받지 않나?" → **아니다.** 보증 수수료의 트리거는 **디폴트가 아니라 투자자의 실현 이익**이다. 보증인은 보험사처럼, "사고가 났냐"가 아니라 "투자자가 이익을 봤냐"를 기준으로 자기 몫을 떼간다.

핵심 공식은 `purchase-fee.service.ts:75` 근처에 있다.

```ts
// 의사 코드 (purchase-fee.service.ts:75 부근)
const totalProfit = redemptionAmount - principal;     // 투자자 1명 기준
const guaranteeFee = totalProfit > 0
  ? totalProfit * guaranteeFeeRate                    // 이익이 있을 때만
  : 0n;                                               // 손실/본전이면 0
```

이걸 시나리오 매트릭스로 풀면 다음과 같다.

| 시나리오 | 투자자 수령 | 투자자 이익 | guaranteeFee | 보증인 입장 |
|---|---|---|---|---|
| 정상 상환 | 원금+이자 | **>0** | `profit × fee%` | 가만히 앉아 수수료만 챙김 (보험사 수익 최대) |
| 디폴트 + 보증 보상 (전액) | 원금+이자 (보증인이 메움) | **>0** | `profit × fee%` | 보상금 지출 > 수수료 가능 (보험금 청구 사건) |
| 디폴트 + 부분 회수 (보증 부족/비보증) | 원금 일부만 | **≤0** | **0** | 수수료도 못 받고 사건만 남음 |
| 보증 없는 invoice 디폴트 | 일부만 또는 0 | **≤0** | **0** | 해당 없음 (보증인 자체가 없음) |

> [!note] 보증인 = 보험사 모델
> 보증인은 "투자자가 돈을 잃지 않게 해주는 대신, 투자자가 번 돈에서 일정 비율을 가져가는" 구조. 손해를 보상해주는데 수수료를 못 받는 케이스(3번)는 보험 사고와 동일한 비대칭이다.

### 2. 실제 청구 시점: `FractionsRedeemed` 이벤트

수수료는 구매 시점이 아니라 **redeem 시점**에 확정된다. 흐름은 다음과 같다.

```
투자자 redeem 트랜잭션 (on-chain)
        │
        ▼
  컨트랙트가 USDC 지급
        │
        ├─ 투자자 지갑 ← redemptionAmount
        └─ Guarantor Vault → 보증인 지갑 ← guarantorFeeUsdc
        │
        ▼
  FractionsRedeemed 이벤트 emit
        │  { investor, principal, redemptionAmount, guarantorFeeUsdc, ... }
        ▼
  ar-service가 이벤트 수신 → DB 기록
```

> [!tip] 오프체인은 "받아 적기"만 한다
> ar-service는 보증 수수료를 **계산해서 지시하지 않는다.** 온체인 컨트랙트가 redemption 로직 안에서 자체 계산해 분배하고, ar-service는 `FractionsRedeemed.guarantorFeeUsdc` 값을 그대로 받아 기록할 뿐이다. 즉 "fee를 받았다"의 source of truth는 **온체인 이벤트**다.

### 3. 여러 투자자가 한 invoice를 나눠 샀을 때

> [!warning] "수수료를 나눈다"는 표현은 부정확
> 한 invoice의 총 수수료를 N등분 하는 게 아니다. **PurchaseTransaction마다 독립 계산**된다. 즉 "나누는" 게 아니라 **애초에 따로따로 계산되는** 구조다.

`purchase-investor.usecase.ts:357` 근처에서 보이듯, 구매 1건 = PurchaseTransaction 1행이고, redeem도 그 구매 단위로 진행된다.

```
Invoice #42 (총 100 조각, 보증 수수료율 = 10%)
│
├─ Alice가 60 조각 매수 → PurchaseTransaction A
│    └─ redeem 시 profit_A = 6 USDC → fee_A = 0.6 USDC
│
├─ Bob이  30 조각 매수 → PurchaseTransaction B
│    └─ redeem 시 profit_B = 3 USDC → fee_B = 0.3 USDC
│
└─ Carol이 10 조각 매수 → PurchaseTransaction C
     └─ redeem 시 profit_C = 1 USDC → fee_C = 0.1 USDC

보증인이 받는 총 fee = 0.6 + 0.3 + 0.1 = 1.0 USDC
(= 총 profit 10 × 10%, 결과적으로는 같지만 "쪼개진" 게 아니라 "각자 계산된" 것)
```

> [!example] 비대칭 결과가 자연스럽게 나온다
> 만약 디폴트가 나서 Alice는 보증 보상으로 원금+이자를 받았는데(이익 발생, fee 청구) Bob의 구매분은 보증 한도 초과로 부분 회수(손실)만 됐다면 → Alice에게서는 fee 청구, Bob에게서는 fee 0. **같은 invoice의 같은 사건이라도 투자자별로 결과가 갈린다.** 조각 = 지분 = 독립 계좌라고 이해하면 된다.

### 4. Partial 시나리오 4종

ar-service에서 "partial"이라는 단어가 의미 있게 등장하는 지점은 4가지다. 같은 단어지만 발생 단계와 처리 방식이 전혀 다르다.

| # | 상황 | 발생 위치 | 시스템 반응 |
|---|---|---|---|
| A | 남은 조각보다 **많이** 사려 함 | 구매 요청 시 | 거부 (HTTP 400) |
| B | 한 invoice가 **일부만** 팔리고 saleEndDate 도달 | 판매 종료 시 | `PARTIALLY_FUNDED`, 계속 판매 |
| C | redeem 시 받을 금액이 **기대치보다 적음** | redeem 시 | `PARTIAL_REDEEM` 마킹 |
| D | 온체인 이벤트와 DB가 **불일치** | 이벤트 검증 시 | `VERIFICATION_FAILED` |

#### A. 초과 구매 거부 (`purchase-investor.usecase.ts:193`)

```ts
// :193 부근
if (purchaseCount > invoice.availableCount) {
  throw new BadRequestException('not enough fractions available');
}
```

남은 게 30 조각인데 50 조각을 사겠다는 요청은 **잘라서 30 조각만 팔지 않는다.** 통째로 거부하고 클라이언트가 다시 요청해야 한다. 부분 체결(partial fill) 개념이 없다.

#### B. 부분 펀딩 → `PARTIALLY_FUNDED` (`purchase-investor.usecase.ts:404` 부근, `get-listings.usecase.ts:75`)

invoice가 100 조각인데 saleEndDate 도달 시점에 60만 팔렸어도 죽지 않는다. 상태가 `LISTED → PARTIALLY_FUNDED`로 바뀌고 마켓에 계속 노출된다 ([[get-listings.usecase.ts]]의 노출 상태: `LISTED / PARTIALLY_FUNDED / SOLD_OUT`).

> [!note] PARTIALLY_FUNDED도 구매 가능 상태
> `PURCHASABLE_INVOICE_STATUSES = [LISTED, PARTIALLY_FUNDED]` (`purchase-investor.usecase.ts:37`). 즉 "조금 팔리다 만" 상품도 마저 살 수 있다.

#### C. `PARTIAL_REDEEM` (`redeem-investor.usecase.ts:280` 부근)

```ts
// redeem-investor.usecase.ts:280 부근
const status = redemptionAmount < expectedAtomic
  ? 'PARTIAL_REDEEM'
  : 'REDEEMED';
```

투자자가 기대했던 만큼(원금+이자)을 다 못 받으면 redeem 자체는 성공한 채 `PARTIAL_REDEEM`으로 마킹된다. 발생 케이스:

- **디폴트 + 보증 한도 부족**: Guarantor Vault에 USDC가 모자라 일부만 보상
- **비보증 invoice의 디폴트**: 채무자(Payer)가 갚은 만큼만 분배

이 경우 위 매트릭스 3번에 해당하여 `guaranteeFee = 0`이 동반된다.

#### D. `VERIFICATION_FAILED`

온체인에서 들어온 이벤트의 조각 수 / 금액이 DB 예상치와 맞지 않을 때 마킹된다. partial이라는 단어를 직접 쓰진 않지만, "기대와 실제가 partial하게 어긋난 상태"라는 점에서 같은 계열의 비정상 경로다.

> [!tip] 한 줄 요약
> 보증 수수료는 **투자자 이익이 양수일 때만 PurchaseTransaction 단위로** 떼이고, partial은 4단계(구매 거부 / 부분 펀딩 / 부분 상환 / 검증 실패) 어디서든 별개의 상태로 표현된다.

## 투자자 100% 현금 수령은 가능한가

결론부터: **현재 코드 베이스로는 불가능**합니다. 투자자의 라이프사이클 전체가 온체인 토큰(주로 [[USDC]]) 기반으로 하드코딩되어 있고, 법정화폐(현금) 수령으로 빠져나갈 수 있는 곁가지가 한 군데도 없습니다. [[채무자]]만 현금이 가능하고, 투자자는 토큰 전용인 **비대칭 설계**입니다.

### 막힌 지점을 코드 레벨로 분해

| # | 게이트 | 위치 | 함의 |
|---|---|---|---|
| ① | `redeem` 진입점이 `txHash` 필수 | `redeem-investor.dto.ts:11` `@IsNotEmpty` | 온체인 트랜잭션 없이는 redeem 호출 자체가 400 |
| ② | 수령액 산출이 `FractionsRedeemed` 이벤트에서 옴 | redeem-investor 유스케이스 | 이벤트가 없으면 redemptionAmount 산출 불가 |
| ③ | USD→USDC **강제 매핑** | `invoice-onchain.service.ts:162` | 통화 표기가 USD여도 실제로는 USDC 토큰 |
| ④ | 구매 자체도 온체인 USDC | `purchase-investor.usecase.ts` | 입장도 출구도 토큰 |
| ⑤ | `Investor` 모델에 은행계좌/IBAN 칼럼 없음 | Prisma 스키마 | 송금할 곳이 없음 |
| ⑥ | `payout` / `disburse` / `bank` / `wire` 키워드 0건 | 전체 레포 grep | 지급 레일 자체가 없음 |

> [!warning] 한 줄 요약
> "투자자가 현금을 받는다"는 분기는 **레이어 1개가 빠진 게 아니라 6개 모두 빠져 있다**. 단순 패치로 끝나는 일이 아닙니다.

### 그림으로 보는 현재 구조의 비대칭

```
                ┌─────────────── 현금 OK ────────────────┐
[Payer/채무자]  ─── 은행 송금/USD/JPY ──▶ [운영자]
                                            │
                                    paybackFiat 기록
                                            │
                                            ▼
                                    Payback Escrow에
                                    USDC로 충당
                                            │
                                            ▼
                ┌─────────── 토큰만 OK (USDC) ───────────┐
                                            │
                                  FractionsRedeemed
                                            │
                                            ▼
                                       [Investor]
                                   (지갑으로 USDC 수령)
                ↑↑ 이 화살표가 fiat로 갈 길이 코드에 없음
```

> [!note] 핵심 통찰
> 채무자 쪽 **현금 ↔ 투자자 쪽 USDC** 사이의 간극은 지금 **운영자의 USDC 충당**으로 메우고 있을 뿐, "fiat → fiat" 직결 레일은 처음부터 설계되지 않았습니다. 통화 분리는 [[currencyOriginal]] / [[listingCurrency]] / [[redemptionCurrency]] 3종으로 표기되지만 마지막 단계는 항상 토큰입니다.

### 방안 A: 오프체인 fiat 정산을 신설

투자자가 **순수 현금만** 받게 만들려면 다음을 전부 신설해야 합니다.

> [!example] 방안 A에 들어가야 할 작업 항목
> - `Investor` 스키마에 **은행계좌 / IBAN / SWIFT / KYC 상태** 칼럼 추가
> - `POST /ar/purchase/redeem-fiat` 신규 엔드포인트 (txHash 면제 분기)
> - 송금 레일 통합: [[Wise]] / [[Stripe Treasury]] / 국내 펌뱅킹 중 택1
> - 정산 상태 머신: `PENDING_FIAT → IN_FLIGHT → SETTLED / FAILED`
> - 환율 동결 시점 및 환차익/환차손 회계 처리
> - **컨트랙트 수정**: 투자자가 NFT/조각을 들고 있는 채로 fiat 정산을 받으려면 컨트랙트가 "redeem 권리 소각만 하고 토큰은 운영자 풀로" 보내는 분기를 알아야 함 → `MusubiARProtocol` 신규 함수

| 영역 | 변경 규모 | 위험도 |
|---|---|---|
| Prisma 스키마 | 중 | 마이그레이션 1회 |
| 신규 엔드포인트/유스케이스 | 대 | 권한·검증 전부 신규 |
| 송금 레일 통합 | 대 | 외부 의존, 컴플라이언스 |
| 컨트랙트 수정 | **상** | 감사·재배포·기존 토큰 호환성 |
| KYC/AML | 중 | 법적 요건 따라 다름 |

> [!warning] 컨트랙트 재배포의 비용
> 이미 발행된 [[MusubiARNFT]]가 있는 상태에서 redeem 분기를 바꾸면 신/구 NFT 간 호환성 / 마이그레이션 / 감사 재실시가 추가됩니다. 컨트랙트 수정이 "거의 필수"인 이유.

### 방안 B: USDC 수령 후 오프램프

기존 구조를 건드리지 않고 투자자만 "결국 현금이 손에 쥐어지게" 만드는 방법입니다.

```
[Investor] ── redeem 그대로 ──▶ USDC 수령 (자기 지갑)
                                     │
                                     ▼
                          오프램프 파트너 ([[Coinbase]] /
                          [[Circle]] / 국내 거래소 KRW 출금)
                                     │
                                     ▼
                              투자자 본인 은행계좌
```

> [!tip] 방안 B가 가벼운 이유
> - ar-service 코어는 **수정 거의 없음** — redeem 흐름은 그대로
> - 컨트랙트 무수정 → 감사·재배포 불요
> - 오프램프는 외부 파트너 책임 영역
> 
> 대신 환율/오프램프 수수료/파트너 KYC가 투자자 경험에 노출되고, 운영자가 "원클릭 fiat"를 약속하려면 오프램프 파트너 API를 안내 페이지에서 연결해 줘야 함.

### 의사결정 매트릭스

| 기준 | 방안 A (fiat 정산 신설) | 방안 B (오프램프) |
|---|---|---|
| 투자자 UX | 완벽한 fiat 네이티브 | 토큰 한 단계 거침 |
| 개발 공수 | 대 (수개월) | 소 (수일~수주) |
| 컨트랙트 변경 | 거의 필수 | 불요 |
| 컴플라이언스 부담 | 운영자가 직접 짊어짐 | 오프램프 파트너에 위임 |
| MVP 적합 | × | ◎ |
| 현금 네이티브 시장 | ◎ | △ |

> [!note] 권장
> - **MVP / 빠른 검증**: 방안 **B** — 코어 무수정, 외부 파트너로 우회
> - **현금 네이티브 시장 (예: 한국 개인 투자자에게 100% KRW 약속) 본격 진입**: 방안 **A** — 다만 컨트랙트 재설계와 송금 레일 통합 비용을 사전 합의

## 다이어그램

[[canvas/ar-service_인보이스_투자_상품_플로우.canvas|개념도]]

---

## English

### TL;DR

`ar-service` invoice investment products tokenize **accounts-receivable (the invoice / right-to-be-paid), not physical vehicles**, as fractional NFTs. The contracts are named `MusubiARProtocol` / `MusubiARNFT` so the [[musubi]] brand is shared, but the IPFS metadata only carries `faceValue` · `maturityDate` · `seller` · `payer` · `contractId` — there are zero VIN/odometer/asset fields anywhere in the repo. On the investor side the lifecycle is hard-wired to **100% on-chain [[USDC]]**: purchase is USDC, maturity redeem is USDC, and the redeem endpoint requires `txHash` — only the debtor (payer) is allowed to settle in fiat, making this an **asymmetric currency design**. The other load-bearing fact is the **staggered timing**: the operator first registers [[Seller]] · [[Payer]] (and [[Guarantor]] if guaranteed) and drives them to `ACTIVE`, then calls `generateInvoices` to mint the note and push it to `LISTED`, and only after the market opens does the [[Investor]] register via `POST /ar/investor` and buy — it is not "all four parties registered up front" but the issuance side and the investment side **joining sequentially**.

> [!note] One-line mental model
> **What is being tokenized?** Receivables, not vehicles. **Whose money?** Investors are USDC-only. **When?** Issuance-side 4-gate clearance → `LISTED` → market opens → investor registers and buys. Those three axes are the skeleton everything else hangs off.

### Binding Target: Receivables, Not Vehicles

In ar-service, the investment product is **not bound to a physical vehicle.** What gets tokenized is "the money to be received for selling the car" — i.e. the [[Account Receivable]] itself. Grep the codebase for `vehicle`, `VIN`, `asset`, `odometer` and you get **zero hits**, and the IPFS metadata for an invoice contains not a single vehicle identifier either.

> [!note] Core takeaway
> ar-service tokenizes a **claim (the right to be repaid)**. Where [[Musubi]] (allegedly) ties a token to the vehicle itself, ar-service only tracks "who owes whom, how much, by when." A vehicle is at best the off-chain backdrop from which the receivable arose; it never enters the system's field of view.

### What goes into IPFS metadata (and what doesn't)

Around `invoice-ipfs.service.ts:80`, when metadata is assembled, exactly these 5 fields get serialized:

| Field | Meaning | Vehicle-related? |
|---|---|---|
| `faceValue` | Face value of the receivable | ✗ |
| `maturityDate` | Maturity date | ✗ |
| `seller` | Originator of the receivable | ✗ |
| `payer` | Debtor who repays at maturity | ✗ |
| `contractId` | Original contract identifier | ✗ |

No VIN, chassis number, model, mileage, vehicle ID — no asset identifier of any kind. One NFT therefore points to a **specific receivable**, not a specific car.

```
[ Physical vehicle ]   ←  Outside ar-service's view
        │
        │ (off-chain sale → receivable arises)
        ▼
[ Receivable (AR) ]    ←  Tokenization target
        │
        │  faceValue / maturityDate / seller / payer / contractId
        ▼
[ IPFS metadata ]  →  [[MusubiARNFT]] mint
```

### Difference from Musubi — shared name, different binding

The contracts are named `MusubiARProtocol`, `MusubiARNFT`, sharing the [[Musubi]] brand, which is easy to misread. But the **binding layer is wholly different.**

| Aspect | [[Musubi]] (presumed) | [[ar-service]] |
|---|---|---|
| Tokenization target | Vehicle itself | Account receivable |
| Identifier | VIN / vehicle ID | `contractId` / `payer` |
| Maturity concept | (none, held asset) | Has `maturityDate` |
| Redemption trigger | Sale/repossession of vehicle | Debtor repayment |
| Meaning of default | Vehicle loss/seizure | Payer fails to repay |

The names look alike, but ar-service's NFT is not "a slice of ownership in this car" — it is **"a fractional share of this receivable."**

### vehicleOracle / musubiService — declared, but called 0 times

`infra.config` declares HTTP slots for `vehicleOracleService` and `musubiService`. A first-time reader easily misreads this as "ah, vehicle prices or status are pulled from an external service." But grep the actual call sites:

```bash
$ grep -rn "vehicleOracleService\." src/
# (no calls outside the declaration)

$ grep -rn "musubiService\." src/
# (no calls outside the declaration)
```

**They are never invoked anywhere in the investment-product lifecycle.** Invoice generation, market opening, purchase, maturity redeem, default handling — none of those paths poke a vehicle oracle.

> [!warning] Trap: a config slot ≠ a real dependency
> Don't infer from the presence of a `vehicleOracleService` slot in env that "something moves based on vehicle prices." It almost certainly has no behavioral effect — it is **legacy/placeholder**, a seat held open for a future vehicle-valuation integration.

### Why this matters

The receivable-binding model rewrites the premises of every other section:

- The **maturity trigger** is the `payer`'s repayment, not a vehicle sale — which is why [[통화/에스크로/만기 수령 구조]] is shaped as "Payer repays USDC → Payback Escrow → investor redeem."
- The **default scenario** is non-repayment, not vehicle loss — which is why [[보증 수수료와 디폴트/부분(partial) 시나리오]] brings in the [[Guarantor Vault]].
- In **participant registration**, the parties registered are `seller`/`payer`/(optionally) `guarantor`, not a vehicle owner — because the system asks "who issues and who repays this receivable," not "who holds this car."

> [!example] One-line summary
> ar-service **tokenizes debt, not cars.** IPFS metadata carries only `faceValue/maturityDate/seller/payer/contractId`, and `vehicleOracle`·`musubiService` are declared but never called.

### Participant Registration and the Meaning of "Start"

Two things matter in this section: ① **which participants must be registered before product creation**, and ② **what "start" literally means at the code level**. The intuitive phrasing "register all four (Buyer / Seller / Payer / Guarantor) and start" is misleading — the actual [[ar-service]] implementation does not work that way.

### Registration = DB ACTIVE immediately + on-chain Registry asynchronously

First, separate the fact that the single word "registration" triggers two distinct operations.

| Step | Location | Timing | Status field |
|---|---|---|---|
| 1) Create domain record | Prisma DB | sync (immediate) | `status = ACTIVE` (`@default(ACTIVE)`) |
| 2) Register on Registry | [[Outbox]] → SC | async | `onchainStatus: PENDING → CONFIRMED` |

```
register(participant)
  ├─ INSERT participant (status=ACTIVE)              ── sync, in tx
  └─ INSERT outbox (op=REGISTER, onchainStatus=PENDING)
        ↓ (worker picks up)
        sendTx(Registry.register(...))
        ↓ (event received)
        UPDATE participant SET onchainStatus=CONFIRMED
```

> [!note] DB ACTIVE is immediate; the chain lags one beat
> Registering Seller/Payer/Guarantor flips DB `status` to `ACTIVE` right away, but reflecting it in the on-chain Registry requires an outbox worker cycle. The reason the `generateInvoices` gate, described below, **only checks DB ACTIVE** is precisely this async gap.

### Only 3 participants actually need to be registered (Buyer excluded)

Grep `generate-invoices.usecase.ts` and you will find **zero** references to `investor` / `buyer`. The preconditions for product (invoice-token) creation are only:

- [[Seller]] — the party selling the receivable (the firm needing cash)
- [[Payer]] — the debtor who will pay the receivable
- [[Guarantor]] — *optional*. Required only for guaranteed products, and even then only as a prerequisite for the `GuaranteePosition` step, not for invoice creation itself.

[[Investor]] (buyer) is *not* a precondition. After the market is open, they register themselves via `POST /ar/investor` (see [[마켓 오픈과 투자자 합류 시점]]).

> [!warning] The "register all 4 and start" phrasing creates a misconception
> Operationally, the buyer is the subject of the *next* phase. To create a product, you only need the two parties to the receivable (Seller↔Payer) plus an optional Guarantor. Investors come in *after* the invoice has been listed, registering themselves on the way in.

### "Start" = calling `generateInvoices`

The precise meaning of "start" is **calling the `generateInvoices` use case to create the invoice (note) records bound to a Contract**. Only after this call succeeds does an invoice exist in `DRAFT` or downstream states, and only then can the rest of the flow (market listing, purchase, redemption at maturity) proceed.

### Four gates

`generateInvoices` requires all four of the following to hold:

```
                 generateInvoices(contractId)
                          │
        ┌─────────────────┼─────────────────┬─────────────────┐
        ▼                 ▼                 ▼                 ▼
 ① Contract.status   ② Contract.payer   ③ Seller.status   ④ Payer.status
    === DRAFT          Confirmation         === ACTIVE        === ACTIVE
                       === RECEIVED         (DB)              (DB)
```

| # | Gate | Meaning |
|---|---|---|
| ① | `Contract.status = DRAFT` | Only contracts not yet issued as invoices are eligible |
| ② | `Contract.payerConfirmation = RECEIVED` | Payer has acknowledged "I owe this" |
| ③ | `Seller.status = ACTIVE` (DB) | Seller registered and operationally active |
| ④ | `Payer.status = ACTIVE` (DB) | Payer registered and operationally active |

> [!tip] The gate only checks **DB ACTIVE** — not `onchainStatus`
> `generateInvoices` does *not* verify `onchainStatus === CONFIRMED` for Seller/Payer. As a result, an invoice `DRAFT` can be created while the on-chain Registry entry is still `PENDING`.
>
> **Why this is usually fine**: by the time the invoice itself needs to be listed on the market or purchased, the invoice's own `onchainStatus = CONFIRMED` is required (`PURCHASABLE_INVOICE_STATUSES = [LISTED, PARTIALLY_FUNDED]` + `onchainStatus = CONFIRMED`). By then the Seller/Payer registration outbox has very likely confirmed.
>
> **Caveat worth knowing**: if you push the invoice's SC registration while Seller/Payer is still PENDING on-chain, the SC call can revert. In practice you want to wait for outbox confirmation before progressing.

### Investor registration has a different shape

For contrast, [[Investor]] registration:

- Prisma model uses `@default(ACTIVE)` — **no approval step**, ACTIVE immediately upon creation
- **No** on-chain Registry registration (no outbox emitted)
- `userId` idempotency check → fetch the main wallet from [[user-service]] (404 if none) → duplicate wallet check (409) → fill in email/name → create
- Main wallet required

So while everyone ends up `ACTIVE`, the ACTIVE of Seller/Payer/Guarantor has *an async tail to the on-chain Registry*, whereas the Investor's ACTIVE is *a single DB write*.

> [!example] One-line operational scenario
> 1) Register Seller → DB ACTIVE immediately, on-chain PENDING → CONFIRMED (async)
> 2) Register Payer → same
> 3) (optional) Register Guarantor → same. Required when attaching a `GuaranteePosition` to a guaranteed product
> 4) Create Contract `DRAFT`; Payer acknowledges → `payerConfirmation = RECEIVED`
> 5) **This is "start" = `generateInvoices(contractId)`** → if all 4 gates pass, invoice DRAFT is created
> 6) Invoice SC registration → `onchainStatus = CONFIRMED` → market `LISTED`
> 7) **Now** investors come in, register themselves, and purchase

### Market Opening and When Investors Join

The previous section said "register Seller/Payer/(Guarantor) first, then create the product," which naturally raises the question: "does the investor (buyer) also need to be pre-registered?" The short answer is **no** — the market opens *after* the product is created, and investors walk in on their own afterward.

### Two phases on the timeline

```
[T0] Operator-side work                 [T1] Market opens          [T2] Investor joins
─────────────────────────              ─────────────────         ─────────────────
- Register Seller (ACTIVE + on-chain)   - invoice status=LISTED   - GET /ar/market/listing
- Register Payer  (ACTIVE + on-chain)   - GET /ar/market/listing  (browse, no auth needed)
- (Register Guarantor + Position)         exposed as Public        ─────────────────
- Create Contract (DRAFT)                                          - POST /ar/investor
- Call generateInvoices()                                            (self-register, get JWT)
- invoice → LISTED + saleStartDate                                 - POST /ar/purchase
                                                                     (JWT AR_INVESTOR)
```

So the shorthand "register all four parties → start" is a bit misleading. Only the **supply-side three (Seller, Payer, +optional Guarantor)** are prerequisites for product creation; the **demand side (Investor)** joins the market with a time gap.

### The auth boundary is baked into the code

The permission decorators on the market controller make these two phases explicit:

| Endpoint | Auth | Purpose |
|---|---|---|
| `GET /ar/market/listing` | **Public** | Anyone, no login — comment says *"investor can connect wallet later"* |
| `GET /ar/market/listing/:id` | **Public** | Single invoice detail |
| `POST /ar/investor` | Public (user JWT) | Register oneself as [[ar-service-investor]] |
| `POST /ar/purchase` | **JWT AR_INVESTOR** | Registered investors only |
| `POST /ar/purchase/redeem` | **JWT AR_INVESTOR** | Registered investors only |

`get-listings.usecase.ts:75` narrows the market-visible statuses to three: `LISTED`, `PARTIALLY_FUNDED`, `SOLD_OUT`. DRAFT and REDEEMED never show up in the market.

> [!note] Asymmetric market access
> **Anyone can browse, only registered investors can buy.** The intended flow is to look around before connecting a wallet, then register via [[POST /ar/investor]] once interested.

### Investor registration = immediately ACTIVE

The `create-investor.usecase` flow is surprisingly thin:

```
POST /ar/investor (user JWT)
   │
   ├─ userId idempotency check (return as-is if already registered)
   ├─ Fetch main wallet from user-service   ─→ 404 if none
   ├─ Wallet uniqueness check                ─→ 409 if another investor owns it
   ├─ Pull email / name and assemble row
   └─ INSERT Investor { status: ACTIVE }     ← Prisma @default(ACTIVE)
```

Key characteristics:

- **No approval workflow** — there's no `PENDING → APPROVED` state machine at all. Prisma's `status` default is literally `ACTIVE`.
- **Main wallet is required** — blocked with 404 if no main wallet is registered in [[user-service]].
- **No on-chain Registry registration** — unlike Seller/Payer/Guarantor, investors don't emit a `REGISTER` outbox event. The [[onchainStatus]] field doesn't even exist on the Investor model. From the contract's point of view, an investor is just "some wallet address holding a note."

> [!tip] Why only investors skip on-chain registration
> Seller/Payer/Guarantor are **subjects bound to and verified by the invoice itself** (issuer, debtor, guarantor), but the investor is merely an **NFT holder**. ERC-721/1155 already verifies "does this wallet hold the token," so no separate whitelist is needed.

### The 5 + 1 purchase gate

Being registered doesn't mean you can buy any invoice — `purchase-investor.usecase.ts` runs six checks on every purchase request:

```ts
const PURCHASABLE_INVOICE_STATUSES = [
  InvoiceStatus.LISTED,
  InvoiceStatus.PARTIALLY_FUNDED,
]; // :37
```

| # | Check | Line | Fail-mode |
|---|---|---|---|
| 1 | `invoice.status ∈ {LISTED, PARTIALLY_FUNDED}` | `:173` | 400 — blocks DRAFT/SOLD_OUT/REDEEMED |
| 2 | `invoice.onchainStatus === CONFIRMED` | `:179` | 400 — even if DB says LISTED, SC mint not done yet |
| 3 | `now ≥ saleStartDate` | `:186` | 400 — before sale opens |
| 4 | `now ≤ saleEndDate` | `:189` | 400 — past cutoff (`saleEndDate = dueDate − cutoffDays`) |
| 5 | `invoice.availableCount ≥ requested fractions` | `:193` | 400 — buying more than what's left |
| +1 | `investor.status === ACTIVE` | `:67` | 403 — unregistered/inactive |

> [!example] Two "actives" must hold simultaneously
> - **Investor active**: `Investor.status === ACTIVE` (satisfied at registration)
> - **Invoice purchasable**: the bundle of conditions 1~5
>
> Either side missing means the purchase fails. Being visible in the market doesn't mean you can buy — the time window (`saleStart ~ saleEnd`) and remaining fractions must both still be alive.

### The time window carved by saleStartDate / saleEndDate

At `generateInvoices` time, `cutoffDays` is subtracted to derive `saleEndDate`:

```
saleStartDate ──────── purchase window ──────── saleEndDate ──── dueDate (maturity)
                                                 │ ←cutoffDays→  │
                                                 │
                                         past here = no new buys,
                                         only existing holders wait for redeem
```

Letting people buy in right before maturity would tangle the redeem accounting, so the market closes early by `cutoffDays` as a safety margin. Even when the market status stays `LISTED`, gate #4 (`now ≤ saleEndDate`) blocks the trade — so "visible" and "buyable" split again on the time axis.

### Putting it together — joining is a one-shot procedure

The investor's join sequence boils down to one line:

```
Connect wallet → POST /ar/investor → instantly ACTIVE → get JWT → POST /ar/purchase
   (user JWT)                         (DB only,           (AR_INVESTOR)   (if 5+1 gate passes,
                                       no on-chain)                        PurchaseTransaction is created)
```

There's no operator approval queue, no KYC waiting line, no on-chain whitelist transaction. "Investor join" is designed to be frictionless and instantaneous; the actual verification weight sits at the **6-gate check at purchase time.**

### Currency / Escrow / Maturity Payout Structure

ar-service splits **currency into three axes**, but actual fund movement converges onto a **single on-chain USDC escrow stack**. The design is asymmetric: only the debtor (Payer) can touch real cash; the investor only ever touches tokens.

#### 1) Three currency fields — `currencyOriginal` / `listingCurrency` / `redemptionCurrency`

Every [[invoice]] carries three currency fields. They look similar but play very different roles.

| Field | Meaning | Audience | Actual use |
|---|---|---|---|
| `currencyOriginal` | Original denomination of the debt (USD, USDC, JPYC) | Accounting / off-chain records | Note metadata, IPFS |
| `listingCurrency` | Currency exposed in the marketplace; the currency investors trade in | Investors | **Key for on-chain token-address mapping** |
| `redemptionCurrency` | Currency shown for maturity payout | Reports | **Effectively unused** in the redeem path |

> [!note] Why three separate fields?
> Debt may originally be in USD but be sold to investors in USDC. Splitting the accounting denomination (`currencyOriginal`) from the tokenized trading denomination (`listingCurrency`) lets the same ledger surface multiple representations. `redemptionCurrency` does not act as a real gate at payout time — the actual payout token is determined by the `listingCurrency` mapping.

#### 2) USD → USDC forced mapping

The most important trick is that **incoming USD is unconditionally substituted with the USDC token address**. The mapping near `invoice-onchain.service.ts:162` is roughly:

```ts
switch (listingCurrency) {
  case 'USDC':
  case 'USD':         // ← USD denominated in cash is also forced to the USDC contract
    return USDC_TOKEN_ADDRESS;
  case 'JPYC':
    return JPYC_TOKEN_ADDRESS;
  default:
    throw new UnsupportedCurrencyError();
}
```

> [!warning] A "cash USD invoice" means **the debtor pays back in cash**, NOT that the investor receives cash.
> The same invoice can be repaid by the Payer via bank wire in USD, but at that moment the operator must inject an equivalent amount of USDC into [[Payback-Escrow]] for the investor to redeem.

#### 3) Three escrows — currency-agnostic, all on-chain USDC

ar-service has **no "cash-only escrow"**. Regardless of `listingCurrency`, funds flow through the same three on-chain contracts.

```
┌──────────────────────────────────────────────────────────────┐
│                    ON-CHAIN (USDC-denominated)               │
│                                                              │
│   [Purchase Escrow] ←──── Investor USDC deposit (on buy)     │
│         │                                                    │
│         │ after generateInvoices / funding completes         │
│         ▼                                                    │
│      Seller wallet (receives advance)                        │
│                                                              │
│   [Payback Escrow] ←──── Payer's repayment USDC, pre-maturity│
│         │                                                    │
│         │ on redeem                                          │
│         ▼                                                    │
│      Investor wallet (receives USDC)                         │
│                                                              │
│   [Guarantor Vault] ←──── Guarantor collateral USDC          │
│         │                                                    │
│         │ on default + guarantee firing                      │
│         ▼                                                    │
│      Payback Escrow / Investor wallet (guarantee payout)     │
└──────────────────────────────────────────────────────────────┘
```

| Escrow | Token | Withdrawal trigger | On default |
|---|---|---|---|
| **Purchase Escrow** | USDC | Funding completion / sale close | Settles to Seller |
| **Payback Escrow** | USDC | Investor `redeem` | Shortfall covered from Vault |
| **Guarantor Vault** | USDC | Default + guarantee fires | Funds move into Payback Escrow |

> [!tip] Currency-agnostic ≠ contract-agnostic
> Different currencies don't spin up separate contracts. Only the `listingCurrency` → token-address mapping changes; the escrow contracts themselves are the same. For a JPYC invoice, the same Escrow simply holds the JPYC token instead.

#### 4) Debtor cash repayment → operator USDC top-up

When the Payer actually repays in **physical cash** (USD/JPY/KRW), the flow takes this asymmetric shape:

```
   Payer (debtor)                          Operator
   ─────────────                          ──────────
   1) Bank wire (real USD/JPY/KRW)  ────▶  Receives physical funds
                                            │
                                            ▼
                                       2) Record `paybackFiat` (DB)
                                            │
                                            ▼
                                       3) Operator transfers USDC
                                          from its treasury into
                                          Payback Escrow
                                            │
                                            ▼
                              ┌──── Payback Escrow (now loaded with USDC)
                              │
                              ▼
                        4) Investor calls redeem
                              │
                              ▼
                        Investor wallet (USDC received)
```

> [!example] Crucially: **the operator carries the FX risk and the bridging burden**
> If the Payer wires 100,000 USD cash, the operator must inject 100,000 USDC from its own holdings into the Payback Escrow. The currency gap between the debtor (cash) and the investor (USDC) is closed by the operator's own capital / USDC reserves.

#### 5) Maturity payout — USDC dispensed via the `FractionsRedeemed` event

There is **exactly one path** for an investor to collect at maturity. The entry point is `POST /ar/purchase/redeem` (purchase.controller.ts:51).

```
  Investor wallet                             On-chain Contract
  ─────────────                               ──────────────────
  Sign & broadcast redeem() tx   ───────────▶ redeem executes
                                                  │
                                                  │ Normal: draws from Payback Escrow
                                                  │ Default + guarantee: draws from Guarantor Vault
                                                  ▼
                                              USDC transfer → Investor wallet
                                                  │
                                                  ▼
                                              Emit `FractionsRedeemed` event
                                                  │
                                                  ▼
  ar-service: ingest event              ◀────── (txHash required, redeem-investor.dto.ts:11)
       │
       ▼
  Record PurchaseTransaction.redemptionAmount
  Subtract guarantorFeeUsdc, compute investor PnL
```

> [!note] `FractionsRedeemed` is the source of truth
> ar-service does **not** compute the maturity payout itself. It takes the `FractionsRedeemed` event payload (e.g. `redemptionAmount`, `guarantorFeeUsdc`) from the chain and writes it down verbatim. The redeem DTO marks `txHash` as `@IsNotEmpty`, so **redeem is structurally impossible without a real on-chain transaction**.

#### 6) The asymmetry, summarized

| Role | Money form | Note |
|---|---|---|
| **Seller** | On-chain USDC (Purchase Escrow → Seller wallet) | Advance payment |
| **Payer (debtor)** | **Cash OK** (USD/JPY/KRW) or USDC | The only party allowed to use cash |
| **Investor (buyer)** | **USDC only** | Buy and redeem both tokenized |
| **Guarantor** | On-chain USDC (Vault deposit) | Collateral and payouts both tokenized |

> [!warning] 100% cash payout to investors is **not possible with the current structure**
> The redeem entrypoint mandates a `txHash`, the payout amount comes from an on-chain event, the Investor model has no bank account / IBAN field, and there are zero off-chain payout/disburse/bankTransfer rails in the codebase. This topic is covered separately in the [[Can investors receive 100% cash?]] section.

### Guarantee fee and default / partial scenarios

This section pins down, in code terms, when the [[Guarantor]]'s fee actually fires in [[ar-service]], how that fee splits when several investors share one [[invoice]], and what each of the four "partial" scenarios actually means.

### 1. The real trigger: investor profit, not default

> [!warning] The most common misconception
> "If a default happened and the guarantor paid out, then surely the guarantor doesn't collect a fee?" → **Wrong.** The trigger for the guarantee fee is **not whether a default occurred**, but **whether the investor realized a positive profit.** The guarantor behaves like an insurance company: the fee is charged based on "did the investor make money", not "was there a claim".

The core formula sits around `purchase-fee.service.ts:75`:

```ts
// pseudo (around purchase-fee.service.ts:75)
const totalProfit = redemptionAmount - principal;     // per investor
const guaranteeFee = totalProfit > 0
  ? totalProfit * guaranteeFeeRate                    // only when profit > 0
  : 0n;                                               // loss or break-even => 0
```

The resulting matrix:

| Scenario | Investor receives | Investor profit | guaranteeFee | Guarantor's view |
|---|---|---|---|---|
| Normal redemption | principal + yield | **>0** | `profit × fee%` | Pure premium income (best case for insurer) |
| Default + full guarantee payout | principal + yield (guarantor-funded) | **>0** | `profit × fee%` | Payout can exceed fee (a claim event) |
| Default + partial recovery (under-collateralized / uncovered) | only part of principal | **≤0** | **0** | Eats the loss and gets no fee either |
| No-guarantee invoice default | partial or 0 | **≤0** | **0** | N/A (no guarantor exists) |

> [!note] Guarantor = insurer
> The guarantor protects the investor's downside in exchange for a slice of the investor's upside. The asymmetric case (covers the loss but earns no fee) is exactly how insurance claims work.

### 2. Where the fee is actually settled: `FractionsRedeemed`

The fee is finalized at **redeem time**, not at purchase time.

```
Investor sends redeem tx (on-chain)
        │
        ▼
  Contract pays out USDC
        │
        ├─ Investor wallet  ← redemptionAmount
        └─ Guarantor Vault  → Guarantor wallet ← guarantorFeeUsdc
        │
        ▼
  FractionsRedeemed event emitted
        │  { investor, principal, redemptionAmount, guarantorFeeUsdc, ... }
        ▼
  ar-service ingests event → writes to DB
```

> [!tip] Off-chain only "writes it down"
> ar-service does **not** compute or instruct the guarantee fee. The on-chain contract calculates and distributes it inside its redemption logic, and ar-service simply records `FractionsRedeemed.guarantorFeeUsdc`. The on-chain event is the source of truth.

### 3. When multiple investors share one invoice

> [!warning] "Split the fee" is the wrong mental model
> The total fee is **not** divided N ways after the fact. Each `PurchaseTransaction` is computed **independently** at `purchase-investor.usecase.ts:357` and onwards. There's nothing to split because the numbers were never pooled.

```
Invoice #42 (100 fractions total, guarantee fee rate = 10%)
│
├─ Alice buys 60 fractions → PurchaseTransaction A
│    └─ on redeem: profit_A = 6 USDC → fee_A = 0.6 USDC
│
├─ Bob   buys 30 fractions → PurchaseTransaction B
│    └─ on redeem: profit_B = 3 USDC → fee_B = 0.3 USDC
│
└─ Carol buys 10 fractions → PurchaseTransaction C
     └─ on redeem: profit_C = 1 USDC → fee_C = 0.1 USDC

Total fee to guarantor = 0.6 + 0.3 + 0.1 = 1.0 USDC
(= 10 total profit × 10%, equal in aggregate but never actually pooled)
```

> [!example] Asymmetric outcomes fall out naturally
> Suppose a default occurs: Alice's slice is fully covered by the guarantor (profit > 0 → fee charged), but Bob's slice exceeds the remaining guarantee cap and only partially recovers (loss → fee = 0). **Same invoice, same event, different outcomes per investor.** Fractions are independent positions, not a shared pool.

### 4. The four "partial" scenarios

The word "partial" shows up in four distinct places in ar-service. Same word, completely different stages and handling:

| # | Situation | Where | Behavior |
|---|---|---|---|
| A | Buying **more than available** | At purchase request | Rejected (HTTP 400) |
| B | Invoice only **partly sold** by saleEndDate | At sale close | `PARTIALLY_FUNDED`, keep selling |
| C | Redeem pays **less than expected** | At redeem time | Marked `PARTIAL_REDEEM` |
| D | On-chain event **disagrees** with DB | At event verification | `VERIFICATION_FAILED` |

#### A. Reject over-purchase (`purchase-investor.usecase.ts:193`)

```ts
// around :193
if (purchaseCount > invoice.availableCount) {
  throw new BadRequestException('not enough fractions available');
}
```

If 30 fractions are left and someone asks for 50, the system does **not** trim and fill 30. The whole request is rejected; the client must resubmit. There is no partial-fill semantics.

#### B. `PARTIALLY_FUNDED` (`purchase-investor.usecase.ts:404` area, `get-listings.usecase.ts:75`)

If only 60 of 100 fractions sell by saleEndDate, the invoice doesn't die. State flips `LISTED → PARTIALLY_FUNDED` and it stays on the market ([[get-listings.usecase.ts]] exposes `LISTED / PARTIALLY_FUNDED / SOLD_OUT`).

> [!note] `PARTIALLY_FUNDED` is still purchasable
> `PURCHASABLE_INVOICE_STATUSES = [LISTED, PARTIALLY_FUNDED]` (`purchase-investor.usecase.ts:37`). Investors can keep buying into a partially-funded invoice.

#### C. `PARTIAL_REDEEM` (`redeem-investor.usecase.ts:280` area)

```ts
// around redeem-investor.usecase.ts:280
const status = redemptionAmount < expectedAtomic
  ? 'PARTIAL_REDEEM'
  : 'REDEEMED';
```

If the investor gets less than (principal + expected yield), redeem still succeeds but is tagged `PARTIAL_REDEEM`. Causes:

- **Default + guarantee cap exhausted**: Guarantor Vault doesn't have enough USDC for a full payout.
- **Default on an uncovered invoice**: Investors receive only what the payer actually paid back.

This is the row-3 case from the matrix above and always pairs with `guaranteeFee = 0`.

#### D. `VERIFICATION_FAILED`

Marked when the on-chain event's fraction count or amounts disagree with the DB's expectation. The word "partial" isn't used directly, but it's the same family of "expected vs actual mismatched" states.

> [!tip] One-liner
> The guarantee fee fires **per `PurchaseTransaction`, only when that investor's profit is positive**, and "partial" is a four-headed label that can mean rejection, partial funding, partial redemption, or verification mismatch depending on the stage.

### Can investors receive 100% cash?

Bottom line: **not possible with the current code base.** The entire investor lifecycle is hard-coded around on-chain tokens (primarily [[USDC]]), and there is not a single side branch that exits to fiat. The system is **asymmetric** — only the [[Payer]] (debtor) can deal in cash; investors are token-only.

### Breaking down the blockers at code level

| # | Gate | Location | Implication |
|---|---|---|---|
| ① | `redeem` endpoint requires `txHash` | `redeem-investor.dto.ts:11` `@IsNotEmpty` | Without an on-chain tx, the redeem call itself 400s |
| ② | Payout amount comes from `FractionsRedeemed` event | redeem-investor usecase | No event → no `redemptionAmount` |
| ③ | USD→USDC **forced mapping** | `invoice-onchain.service.ts:162` | "USD" currency label still maps to the USDC token |
| ④ | Purchase itself is also on-chain USDC | `purchase-investor.usecase.ts` | Both entry and exit are tokens |
| ⑤ | `Investor` model has no bank-account / IBAN column | Prisma schema | Nowhere to wire money to |
| ⑥ | Zero hits for `payout` / `disburse` / `bank` / `wire` | full-repo grep | The disbursement rail itself does not exist |

> [!warning] One-liner
> "Investor receives cash" isn't **missing one layer — it's missing all six**. This is not a small patch.

### The current asymmetry visualised

```
                ┌─────────────── cash OK ────────────────┐
[Payer/debtor]  ── bank wire / USD / JPY ──▶ [Operator]
                                                │
                                       record paybackFiat
                                                │
                                                ▼
                                       Top up Payback Escrow
                                       with USDC
                                                │
                                                ▼
                ┌────────── tokens only (USDC) ──────────┐
                                                │
                                       FractionsRedeemed
                                                │
                                                ▼
                                            [Investor]
                                       (wallet receives USDC)
                ↑↑ no code path here goes back to fiat
```

> [!note] Key insight
> The gap between **debtor-side cash ↔ investor-side USDC** is currently bridged only by the **operator topping up with USDC**. A direct "fiat → fiat" rail was never designed. Currency separation exists as [[currencyOriginal]] / [[listingCurrency]] / [[redemptionCurrency]], but the final hop is always a token.

### Option A: Build out an off-chain fiat settlement

To let investors receive **pure cash**, all of the following must be newly built:

> [!example] What Option A demands
> - Add **bank account / IBAN / SWIFT / KYC status** columns to the `Investor` schema
> - New `POST /ar/purchase/redeem-fiat` endpoint (txHash-exempt branch)
> - Wire-rail integration: pick one of [[Wise]] / [[Stripe Treasury]] / domestic firm banking
> - Settlement state machine: `PENDING_FIAT → IN_FLIGHT → SETTLED / FAILED`
> - FX-rate freeze timing and FX gain/loss accounting
> - **Contract modification**: if investors hold NFT fractions while being paid fiat, the contract must learn a branch that "burns the redeem right but routes the token to an operator pool" → new function on `MusubiARProtocol`

| Area | Change size | Risk |
|---|---|---|
| Prisma schema | Medium | One migration |
| New endpoint/usecase | Large | Auth + validation all new |
| Wire-rail integration | Large | External deps, compliance |
| Contract modification | **High** | Audit, redeploy, legacy compatibility |
| KYC/AML | Medium | Jurisdiction-dependent |

> [!warning] Cost of contract redeploy
> If [[MusubiARNFT]]s are already issued, changing the redeem branch adds new/old NFT compatibility, migration, and re-audit work. This is why a contract change is "essentially required."

### Option B: Receive USDC, then off-ramp

Without touching the existing structure, make sure cash ends up in the investor's hands:

```
[Investor] ── redeem as-is ──▶ receives USDC (own wallet)
                                       │
                                       ▼
                            Off-ramp partner ([[Coinbase]] /
                            [[Circle]] / domestic exchange KRW
                            withdrawal)
                                       │
                                       ▼
                              Investor's own bank account
```

> [!tip] Why Option B is light
> - ar-service core needs **almost no changes** — redeem flow stays as is
> - No contract change → no audit/redeploy
> - Off-ramp lives in an external partner's domain
>
> The trade-off: FX rates, off-ramp fees, and partner KYC become visible in the investor UX, and if the operator wants to advertise "one-click fiat," they have to wire the off-ramp partner API into the onboarding page.

### Decision matrix

| Criterion | Option A (build fiat settlement) | Option B (off-ramp) |
|---|---|---|
| Investor UX | Fully fiat-native | One token hop |
| Engineering effort | Large (months) | Small (days to weeks) |
| Contract change | Essentially required | None |
| Compliance burden | Operator-borne | Delegated to partner |
| MVP fit | × | ◎ |
| Cash-native market | ◎ | △ |

> [!note] Recommendation
> - **MVP / quick validation**: Option **B** — leave the core untouched and route through an external partner.
> - **Serious push into cash-native markets** (e.g. promising 100% KRW to Korean retail investors): Option **A** — but align upfront on the cost of contract redesign and wire-rail integration.

## Diagram

[[canvas/ar-service_인보이스_투자_상품_플로우.canvas|Concept map]]
