---
id: 019ef75b-e0aa-754a-a9a4-34b642b98aca
title: OIDC와 OAuth 2.0 완벽 이해
topics:
  - oidc
  - oauth2
  - jwt
  - id-token
  - access-token
  - jwks
  - discovery
  - token-rotation
tags:
  - 인증
  - authentication
  - oidc
  - oauth
  - jwt
  - security
sources:
  - 019eed35-942b-733f-880c-ed122611a267
created_at: '2026-06-24T02:00:54.185Z'
updated_at: '2026-06-24T02:00:54.185Z'
---
## TL;DR

[[OAuth 2.0]]은 **인가(Authorization)** — "이 앱이 내 리소스에 접근해도 되는가?" — 를, [[OIDC]]는 그 위에 얹힌 **인증(Authentication)** — "이 사용자는 누구인가?" — 을 담당한다. 두 프로토콜은 거의 같은 [[Authorization Code Flow]]를 공유하지만, 결과물이 다르다.

> [!note] 한 줄 정의
> **OIDC = OAuth 2.0 + ID Token + 표준화된 사용자 정보 조회**. OAuth는 권한을, OIDC는 신원을 발급한다.

| 구분 | [[Access Token]] | [[ID Token]] |
|---|---|---|
| 정의 주체 | [[OAuth 2.0]] | [[OIDC]] |
| 목적 | API/리소스 접근 권한 | 사용자 신원 증명 |
| 형식 | 불투명(opaque) 또는 [[JWT]] (명세상 자유) | **반드시 [[JWT]]** |
| 소비자 | Resource Server | Client(RP) |
| 들여다봐도 되나 | ❌ (클라이언트는 내용을 파싱하면 안 됨) | ✅ (검증 후 sub/email 등 사용) |

```
┌─────────────────────────────────────────────────────┐
│   scope=drive.readonly         → Access Token       │  ← OAuth 2.0
│   scope=openid profile email   → Access + ID Token  │  ← OIDC
└─────────────────────────────────────────────────────┘
```

[[JWT]]는 `Header.Payload.Signature` 3부분으로 구성된다. 안전성의 핵심은 **암호화가 아닌 서명**이다 — Header/Payload는 단순 Base64URL이라 누구나 디코드할 수 있고, JWT가 보장하는 것은 기밀성이 아니라 **무결성(Integrity)**과 **진위성(Authenticity)**이다. [[OIDC]]는 [[RS256]](비대칭) 기본으로, [[OpenID Provider]]가 **개인키로 서명**하고 클라이언트는 [[JWKS]]에서 받은 **공개키로 검증**한다. 개인키 없이는 유효 서명을 만들 수 없고 공개키로는 검증만 가능하므로 위조가 원천 차단된다.

> [!warning] [[Bearer Token]]의 근본 약점
> "토큰 = 권한"이라 **탈취되면 원 주인 행세가 가능**하다. 서명은 위변조는 막지만 "통째로 복사해 쓰는 것"은 막지 못한다. 탈취된 토큰은 완벽하게 유효한 진짜 토큰이다.

방어는 한 가지 기법이 아니라 여러 겹의 합이다:

```
탈취 자체 차단      HTTPS, HttpOnly Cookie, SameSite, XSS 방어
피해 시간 최소화    짧은 Access Token 수명 (5분~1시간)
못 쓰게 만들기      DPoP, mTLS (토큰을 키에 바인딩)
빠른 무효화·탐지    [[Refresh Token Rotation]] + Reuse Detection
```

[[Refresh Token Rotation]]은 RT를 **일회용**으로 만들어, 폐기된 RT가 다시 등장하면 "두 사람이 같은 체인을 쓰고 있다 = 탈취"로 판정하고 해당 [[Token Family]] 전체를 무효화한다. 키 자체도 회전한다 — [[OpenID Provider]]는 [[JWKS]]에 여러 공개키를 두고 토큰 Header의 [[kid]]로 매칭하므로, 키 교체 시 앱은 코드 수정 없이 자동 대응한다. 그리고 이 모든 엔드포인트·키 위치·issuer는 [[OIDC Discovery]] 문서(`/.well-known/openid-configuration`) 하나로 자동 발견되므로, 앱이 알아야 할 것은 **issuer URL 하나뿐**이다.

> [!tip] 기억할 비유
> [[OIDC Discovery]] = OP의 **사용설명서/안내데스크** · `jwks_uri` = **열쇠보관함 주소** · [[JWKS]] = **열쇠보관함** · [[kid]] = **열쇠 이름표**.

## OAuth 2.0 vs OIDC: 인가와 인증의 분리

이 둘은 자주 한 덩어리로 묶여 불리지만, **해결하는 문제가 다르다.** [[OAuth 2.0]]은 "이 앱이 내 리소스를 만져도 되는가"라는 **인가(Authorization)** 의 프로토콜이고, [[OIDC]] (OpenID Connect)는 그 위에 "그래서 이 사용자는 도대체 누구인가"라는 **인증(Authentication)** 을 얹은 확장이다.

### 한 줄로 정리하면

> [!note] 핵심 등식
> **OIDC = [[OAuth 2.0]] + [[ID Token]](신원 정보, JWT) + 표준화된 사용자 정보 조회 방법**
>
> OIDC는 OAuth 2.0을 **대체하지 않는다.** OAuth 2.0의 흐름은 그대로 두고, `scope=openid`를 추가하면 응답에 [[ID Token]]이 하나 더 끼어 나오는 **확장 레이어**다.

### 인가 vs 인증, 한국어로 다시 새기기

| 구분 | OAuth 2.0 | OIDC |
|------|-----------|------|
| 목적 | **인가** (Authorization) | **인증** (Authentication) |
| 답하는 질문 | "이 앱이 내 리소스에 접근해도 되는가?" | "이 사용자는 누구인가?" |
| 결과물 | [[Access Token]] (+ Refresh Token) | [[ID Token]] + Access Token (+ Refresh Token) |
| 토큰 형식 | Access Token은 형식 무규정 (opaque/JWT 모두 가능) | ID Token은 **반드시 JWT** (명세 강제) |
| 대표 사례 | "내 앱이 Google Drive 읽어 가게 해줘" | "Google 계정으로 로그인" |
| 발표 시점 | 2012 (RFC 6749) | 2014 (OpenID Foundation) |

> [!tip] 직관적인 비유
> - **OAuth 2.0** = "호텔 객실 키카드 발급" — 키카드를 들고 있으면 그 방에 들어갈 권한이 있다. **누구의 키카드인지는 카드가 말해주지 않는다.**
> - **OIDC** = "체크인 시 신분증 확인 + 키카드 발급" — 신분증([[ID Token]])으로 "누구인지"를 증명하고, 키카드([[Access Token]])로 "어디에 들어갈 수 있는지"를 받는다.

### 왜 OIDC가 필요했나: Access Token만으로는 신원 위조가 가능하다

OAuth 2.0의 [[Access Token]]은 설계상 **'이 토큰을 가진 사람은 이 권한을 갖는다'** 라는 한 가지 의미만 가진다. 그래서 다음과 같은 **혼동된 대리자(confused deputy)** 문제가 터졌다.

> [!warning] OAuth 2.0을 로그인에 쓰면 생기는 신원 위조
> 과거 많은 서비스가 "Facebook 로그인"을 만들 때 OAuth 2.0의 Access Token을 그대로 신원 증명으로 썼다. 흐름은 이랬다.
>
> 1. 클라이언트 앱이 사용자에게서 `access_token`을 받음
> 2. 앱이 `GET /me?access_token=...` 같은 비표준 엔드포인트로 사용자 ID를 조회
> 3. 그 ID로 로그인 처리
>
> **공격 시나리오:** 공격자가 자기 앱(악성 앱 X)으로 발급받은 Access Token을 정상 앱 Y에 그대로 들이민다. 앱 Y는 그 토큰으로 `/me`를 호출하고 "사용자 A입니다"라는 응답을 받고는 사용자 A로 로그인시킨다. **그러나 그 토큰은 앱 X 용으로 발급된 것이었다.** Access Token에는 *발급 대상(audience)이 누구인지* 검증 가능한 정보가 없기 때문이다. 이를 [[Token Substitution Attack]] 또는 confused deputy 문제라고 부른다.

[[OIDC]]는 이를 정면으로 해결한다. [[ID Token]]은 JWT로, 다음 두 클레임을 **암호학적으로 검증 가능한 방식으로** 포함한다.

- `iss` (issuer): 이 토큰을 누가 발급했는가 (예: `https://accounts.google.com`)
- `aud` (audience): 이 토큰이 **누구를 위해** 발급됐는가 (예: 내 앱의 `client_id`)

내 앱은 `aud`가 자신의 `client_id`와 같은지를 검증하고, 서명을 [[JWKS]] 공개키로 검증한다. 이 둘이 통과되어야만 "이 토큰은 진짜 OP가 우리 앱을 위해 발급한 것"이라고 신뢰할 수 있다.

```
OAuth 2.0만 쓸 때 (위험):
  앱X용 Access Token ──> 앱Y의 /login 엔드포인트
                                    │
                                    └─> "이 토큰의 주인이 누군지" 알 방법 X
                                        그냥 토큰 들고 온 사람=사용자A로 처리 → 위조 성공

OIDC를 쓸 때 (안전):
  앱Y용 ID Token ──> 앱Y가 검증:
                       - 서명 OK? (JWKS 공개키)
                       - iss == "https://accounts.google.com"?
                       - aud == "앱Y의 client_id"?    ← 다른 앱 토큰 거부됨
                       - exp 안 지났음?
                       - nonce 일치?
                     모두 통과 → "이 사람은 사용자A가 맞다" 확정
```

> [!example] 흐름은 거의 같다, 무엇을 요청하느냐만 다르다
>
> **OAuth 2.0 (Google Drive 읽기 권한 위임):**
> ```
> GET /authorize?response_type=code
>                &client_id=...
>                &scope=https://www.googleapis.com/auth/drive.readonly
>                &redirect_uri=...
>                &state=...
> → 결과: Access Token (Drive API 호출용)
> ```
>
> **OIDC (Google 계정으로 로그인):**
> ```
> GET /authorize?response_type=code
>                &client_id=...
>                &scope=openid profile email     ← openid가 OIDC 스위치
>                &redirect_uri=...
>                &state=...
>                &nonce=...                       ← OIDC에서 추가
> → 결과: ID Token + Access Token
> ```
>
> 차이는 단 두 줄. **`scope`에 `openid`가 있는가**, 그리고 응답에 [[ID Token]]이 끼어 오는가. 그래서 OIDC는 "OAuth에 살짝 얹은 인증 레이어"라고 부른다.

### 어느 쪽을 써야 하는가

> [!tip] 의사결정 룰
> - **로그인이 필요하다** (사용자를 식별해야 한다) → [[OIDC]]
> - **다른 서비스의 API를 사용자 권한으로 호출해야 한다** (Drive 읽기, Calendar 쓰기 등) → [[OAuth 2.0]]
> - **둘 다 필요하다** ("Google로 로그인"하고 동시에 Drive도 읽고 싶다) → **OIDC 흐름 한 번**으로 ID Token과 Access Token을 동시에 받는다

소셜 로그인, [[SSO]], B2B 인증 같은 "사용자 식별이 본질인 시나리오"에서는 거의 무조건 OIDC를 쓴다. 반대로 외부 API 위임이 본질인 경우에는 그냥 OAuth 2.0만 써도 충분하다 — 하지만 어차피 사용자 신원도 알고 싶은 경우가 대부분이라, 실무에서 마주치는 대부분의 "소셜 로그인" 흐름은 사실상 OIDC다.

## OAuth 2.0 Authorization Code Flow

OAuth 2.0의 Authorization Code Flow는 "내 앱이 사용자를 대신해서 사용자의 [[google-drive|Google Drive]]에 접근하려면 어떻게 해야 하는가?"라는 질문에 답하는 흐름이다. 핵심은 **사용자 비밀번호를 앱이 절대 보지 않은 채로, 제한된 권한의 토큰만 받아오는 것**이다.

### 4명의 등장인물

먼저 역할을 정확히 구분해야 한다. OAuth 2.0 명세는 다음 4가지 역할을 정의한다.

| 역할 | 누구 | 역할 설명 | 예시 |
|------|------|----------|------|
| **Resource Owner** | 사용자 본인 | 리소스(데이터)의 실제 주인. 권한을 위임하는 사람 | 나(Google 계정 보유자) |
| **Client** | 클라이언트 앱 | 사용자 대신 리소스에 접근하려는 서드파티 앱 | 내가 만드는 백업 앱 |
| **Authorization Server** | 인증/인가 서버 | 사용자를 로그인시키고 동의를 받고 토큰을 발급 | accounts.google.com |
| **Resource Server** | 리소스 API 서버 | Access Token을 검증하고 실제 데이터를 반환 | www.googleapis.com/drive |

> [!note] Authorization Server와 Resource Server는 같은 회사가 운영하지만 명세상 별도 역할이다
> Google의 경우 둘 다 Google이 운영하지만, 토큰 발급 책임(AS)과 데이터 제공 책임(RS)이 분리돼 있다. 그래서 한 AS에서 발급된 토큰으로 여러 RS(Drive, Gmail, Calendar)에 접근할 수 있는 구조가 가능하다.

### 전체 흐름 한눈에 보기

```
┌─────────┐         ┌────────┐         ┌──────────────┐         ┌──────────────┐
│  User   │         │ Client │         │ Auth Server  │         │ Resource     │
│ (Owner) │         │  App   │         │  (Google)    │         │ Server (API) │
└────┬────┘         └───┬────┘         └──────┬───────┘         └──────┬───────┘
     │   ① "Drive 연동"   │                    │                        │
     │ ─────────────────> │                    │                        │
     │                    │  ② /authorize 리다이렉트                    │
     │                    │  client_id, redirect_uri,                   │
     │                    │  scope=drive.readonly, state, PKCE          │
     │ <─────────────────────────────────────  │                        │
     │   ③ 로그인 + 동의 화면                  │                        │
     │ <────────────────────────────────────>  │                        │
     │                    │                    │                        │
     │                    │  ④ authorization code (일회용, ~10분)        │
     │                    │ <───── redirect_uri?code=AUTH_CODE ──────── │
     │                    │                    │                        │
     │                    │  ⑤ POST /token                              │
     │                    │  code + client_id + client_secret           │
     │                    │ ─────────────────> │                        │
     │                    │                    │                        │
     │                    │  ⑥ Access Token (+ Refresh Token)           │
     │                    │ <───────────────── │                        │
     │                    │                    │                        │
     │                    │  ⑦ GET /drive/files                         │
     │                    │  Authorization: Bearer <AccessToken>        │
     │                    │ ──────────────────────────────────────────> │
     │                    │                    │                        │
     │                    │  ⑧ 파일 목록 JSON                            │
     │                    │ <────────────────────────────────────────── │
```

### 단계별 자세히

#### ① 사용자가 연동 버튼 클릭
사용자가 "Google Drive 연동" 같은 버튼을 누르면 클라이언트 앱이 흐름을 시작한다.

#### ② /authorize 리다이렉트 (Front Channel)
앱은 사용자의 브라우저를 [[authorization-endpoint|Authorization Server의 /authorize]]로 리다이렉트한다. 여기서 핵심은 **앱이 직접 호출하는 게 아니라 브라우저를 통해 보내는 것**이다.

```
https://accounts.google.com/o/oauth2/v2/auth?
  response_type=code
  &client_id=12345.apps.googleusercontent.com
  &redirect_uri=https://myapp.com/callback
  &scope=https://www.googleapis.com/auth/drive.readonly
  &state=xyzABC123             ← CSRF 방지용 랜덤값
  &code_challenge=...          ← PKCE
  &code_challenge_method=S256
```

> [!example] scope=drive.readonly의 의미
> `scope`는 앱이 요청하는 권한의 범위다. `drive.readonly`는 "사용자의 Drive 파일을 **읽기만** 할 수 있게 해달라"는 요청이다. `drive`(전체 쓰기 포함)나 `drive.file`(앱이 만든 파일만)과 다르다. 사용자는 동의 화면에서 정확히 이 scope만 허용한다.

#### ③ 사용자 로그인 + 동의 화면
브라우저는 Google 로그인 페이지를 보여준다. 사용자가 로그인하면 "이 앱이 당신의 Drive 파일을 **읽으려고** 합니다. 허용하시겠습니까?" 동의 화면이 뜬다.

> [!tip] 비밀번호는 Google 도메인에서만 입력된다
> 이 단계가 OAuth의 핵심 가치다. 사용자는 `accounts.google.com`에서 비밀번호를 입력하지 ─ 절대 [[client|Client App]]에 비밀번호를 주지 않는다. 그래서 앱이 악의적이어도 비밀번호는 안전하다.

#### ④ Authorization Code 발급
동의가 끝나면 Auth Server는 사용자의 브라우저를 `redirect_uri`로 다시 리다이렉트하면서 `code`를 붙여준다.

```
https://myapp.com/callback?code=4/0AeaY...AUTH_CODE&state=xyzABC123
```

이 `code`는:
- **일회용** (한 번 쓰면 폐기)
- **단명** (보통 10분)
- **앞단(브라우저)을 거쳐 전달되지만 토큰 자체는 아님** → 탈취돼도 client_secret 없이는 토큰 교환 불가

앱은 받은 `state`가 ②에서 보낸 값과 같은지 검증해서 CSRF를 방어한다.

#### ⑤ /token 교환 (Back Channel)
이제 **앱의 백엔드가 직접** Auth Server의 `/token`을 호출한다. 이 호출은 브라우저를 거치지 않는다.

```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=4/0AeaY...AUTH_CODE
&client_id=12345.apps.googleusercontent.com
&client_secret=GOCSPX-xxxxx            ← 백엔드만 알고 있는 비밀
&redirect_uri=https://myapp.com/callback
&code_verifier=...                      ← PKCE
```

> [!warning] code_secret은 절대 프론트엔드에 노출하면 안 된다
> `client_secret`이 노출되면 누구나 그 앱인 척 토큰 교환을 할 수 있다. 그래서 SPA/모바일 같은 public client는 client_secret을 쓸 수 없고 대신 [[pkce|PKCE]]로 보강한다.

#### ⑥ Access Token (+ Refresh Token) 수령
Auth Server는 토큰을 JSON으로 반환한다.

```json
{
  "access_token": "ya29.a0AfH6...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "1//0gK7...",
  "scope": "https://www.googleapis.com/auth/drive.readonly"
}
```

- **Access Token**: API 호출용 (수명 짧음, ~1시간)
- **Refresh Token**: Access Token 갱신용 (수명 김, 며칠~몇 주)
- **scope**: 실제로 부여된 권한 (요청한 것과 다를 수 있음)

#### ⑦ Access Token으로 API 호출
이제 앱은 받은 Access Token으로 Resource Server에 요청을 보낸다.

```http
GET /drive/v3/files HTTP/1.1
Host: www.googleapis.com
Authorization: Bearer ya29.a0AfH6...
```

`Bearer`는 "이 토큰을 들고 온 자에게 권한을 부여한다"는 의미다. Resource Server는 토큰을 검증하고(JWT면 서명 검증, 불투명 토큰이면 AS에 introspection) scope에 맞는 데이터를 돌려준다.

#### ⑧ 리소스 반환
Drive API는 파일 목록을 JSON으로 반환한다. 끝.

### Front Channel vs Back Channel

이 흐름이 안전한 핵심 이유는 **민감한 정보가 브라우저(앞단)를 지나가지 않는다**는 점이다.

| 채널 | 무엇이 지나가나 | 위험 |
|------|---------------|------|
| **Front Channel** (브라우저 리다이렉트) | client_id, scope, state, **authorization code** | code는 일회용 + client_secret 없으면 무용지물 |
| **Back Channel** (앱 백엔드 ↔ Auth Server 직통) | client_secret, **Access Token**, **Refresh Token** | TLS로 암호화, 외부 노출 없음 |

> [!note] Implicit Flow가 폐기된 이유가 바로 이것
> 옛 Implicit Flow는 Access Token을 곧바로 redirect_uri에 붙여서 보냈다(`#access_token=...`). Token이 브라우저 히스토리/Referer로 새는 사고가 잦았고, 이제 OAuth 2.1에서는 완전히 제거됐다. 모든 클라이언트는 **Authorization Code + [[pkce|PKCE]]**를 써야 한다.

### 핵심 통찰

> [!tip] OAuth 2.0의 결과물은 "권한"이지 "신원"이 아니다
> 이 흐름이 끝나면 앱은 "이 Access Token으로 drive.readonly 권한 행사가 가능하다"는 사실만 안다. **그 토큰을 누가 받았는지(사용자가 누구인지)는 모른다.** 이 한계가 OIDC가 등장한 이유이며, OIDC는 동일한 흐름에 `scope=openid`를 추가하고 [[id-token|ID Token]]을 하나 더 받게 해서 이 공백을 메운다 ([[oidc-flow|OIDC 흐름]] 참조).

## OIDC 흐름과 토큰의 종류

[[OIDC]] 흐름은 [[OAuth 2.0]]의 [[Authorization Code Flow]]를 거의 그대로 따른다. 단 하나의 결정적 차이는 — `scope` 파라미터에 **`openid`가 포함되어 있는지** 여부다. 이 한 단어가 들어가는 순간, [[Authorization Server]]는 자신의 역할을 "권한을 발급하는 곳"에서 "신원을 증명해 주는 곳([[OpenID Provider|OP]])"으로 확장하고, 응답에 [[ID Token]]을 하나 더 끼워 넣어 돌려준다.

### 1. OAuth 흐름 위에 `openid`만 얹은 모양

같은 흐름인데 요청 한 줄과 응답 한 줄이 다르다.

```
[OAuth 2.0 — 인가만]
GET /authorize?
    response_type=code
    &client_id=app123
    &redirect_uri=https://app.example/cb
    &scope=drive.readonly                ← OAuth 전용 스코프
    &state=xyz

→ /token 교환 →

{ "access_token": "ya29.a0Af...", "refresh_token": "1//0g..." }
   └── ID Token 없음. "이 사용자가 누구인지"는 알 수 없다.


[OIDC — 인가 + 인증]
GET /authorize?
    response_type=code
    &client_id=app123
    &redirect_uri=https://app.example/cb
    &scope=openid profile email          ← openid 한 단어가 OIDC를 켠다
    &state=xyz
    &nonce=n-0S6_WzA2Mj                  ← replay 방지 (OIDC 필수)

→ /token 교환 →

{
  "access_token":  "ya29.a0Af...",       ← OAuth가 정의한 것
  "id_token":      "eyJhbGciOi...",      ← OIDC가 추가로 얹어준 것 (JWT)
  "refresh_token": "1//0g...",
  "token_type":    "Bearer",
  "expires_in":    3600
}
```

> [!note] 핵심 한 줄
> **OIDC = OAuth 2.0 흐름에 `scope=openid`를 추가하고, 그 결과로 [[ID Token]]을 하나 더 받는 것.** 새 프로토콜이 아니라 OAuth 위에 얹힌 인증 레이어다.

### 2. 세 토큰의 정체와 책임 분리

`/token` 응답에 들어 있는 세 가지는 **각자 발급 목적과 형식이 다르다.** 헷갈리지 않게 한 번에 정리한다.

| 토큰 | 정의한 표준 | 목적 | 형식 | 누가 읽나 | 수명 |
|---|---|---|---|---|---|
| **[[Access Token]]** | [[OAuth 2.0]] | API(리소스) 접근 권한 | **명세상 자유** — 불투명(opaque) 문자열일 수도, [[JWT]]일 수도 있다 | [[Resource Server]] | 짧음 (5분~1시간) |
| **[[ID Token]]** | [[OIDC]] | "이 사용자가 누구인지" 증명 | **항상 [[JWT]]** (명세가 강제) | [[Relying Party\|RP]] (클라이언트 앱) | 짧음 (보통 Access Token과 비슷) |
| **[[Refresh Token]]** | [[OAuth 2.0]] | Access Token 갱신 | 보통 불투명 | [[Authorization Server]] | 김 (며칠~몇 주) |

> [!warning] 자주 혼동하는 두 가지
> 1. **"ID Token으로 JWT를 사용한다"는 부정확하다.** ID Token은 *그 자체가 JWT다* — 형식 선택지가 없다. OIDC 명세가 그렇게 못박았기 때문이다.
> 2. **Access Token은 JWT일 수도, 아닐 수도 있다.** [[Google]]의 Access Token(`ya29.a0Af...`)은 불투명 문자열이고, [[Auth0]]/[[Keycloak]]의 Access Token은 보통 JWT다. **클라이언트는 Access Token의 내용을 열어 보면 안 된다** — Access Token은 Resource Server에게 던지라고 있는 것이지, 클라이언트가 파싱할 대상이 아니다.

각 토큰이 향하는 곳을 그림으로 보면 분리가 명확해진다.

```
                      ┌──────────────────────────┐
                      │   Authorization Server   │
                      │   (OP, 예: Google)        │
                      └────────────┬─────────────┘
                                   │ /token 응답
                  ┌────────────────┼────────────────┐
                  ▼                ▼                ▼
           ┌────────────┐   ┌────────────┐   ┌──────────────┐
           │ ID Token   │   │AccessToken │   │RefreshToken  │
           │ (JWT)      │   │(opaque/JWT)│   │(opaque)      │
           └─────┬──────┘   └─────┬──────┘   └──────┬───────┘
                 │                │                 │
                 ▼                ▼                 ▼
        앱(Relying Party)  Resource Server     Authorization
        가 자체 검증·파싱   (Google Drive API)  Server에만 다시
        해서 "이 사용자가   가 검증             돌려보내 새 Access
        누구"인지 확인                          Token으로 교환
```

> [!tip] 책임이 다르면 검증 주체도 다르다
> [[ID Token]]은 **앱이 직접 검증**한다(서명 + iss/aud/exp/nonce). [[Access Token]]은 **앱이 아니라 Resource Server가 검증**한다. 그래서 클라이언트는 Access Token이 JWT여도 굳이 열어볼 이유가 없다 — 본인 앞으로 온 편지가 아니기 때문이다.

### 3. ID Token 필수 클레임 — `iss / sub / aud / exp / iat / nonce`

[[ID Token]]은 [[JWT]]이므로 `Header.Payload.Signature` 세 부분으로 이루어지며, Payload(클레임 집합)에는 OIDC가 **반드시 들어가야 한다고 못박은 6개 필수 클레임**이 있다.

```json
// ID Token payload 예시 — base64url 디코드한 결과
{
  "iss":   "https://accounts.google.com",          // 발급자 (Issuer)
  "sub":   "10769150350006150715113082",            // 사용자 고유 ID (Subject)
  "aud":   "app123.apps.googleusercontent.com",     // 대상 = 내 client_id (Audience)
  "exp":   1750800000,                               // 만료 시각 (Expiration)
  "iat":   1750796400,                               // 발급 시각 (Issued At)
  "nonce": "n-0S6_WzA2Mj",                           // 요청 시 보냈던 nonce

  // 아래는 scope에 따라 선택적으로 들어오는 클레임
  "email":          "justin.seo@mvlchain.io",
  "email_verified": true,
  "name":           "Justin Seo",
  "picture":        "https://lh3.googleusercontent.com/..."
}
```

각 클레임이 **왜 필수인지** — 빠지면 무엇이 깨지는지로 외우는 게 좋다.

| 클레임 | 의미 | 검증 시 하는 일 | 빠지면? |
|---|---|---|---|
| `iss` | Issuer. 토큰을 누가 발급했는가 | 내가 신뢰하는 OP의 issuer 문자열과 정확히 일치하는지 확인 | 다른 OP가 만든 토큰을 받아들일 위험 |
| `sub` | Subject. OP 내에서 변하지 않는 사용자 고유 ID | DB의 사용자 식별자로 사용 | 동일 사용자 식별 불가 |
| `aud` | Audience. 이 토큰의 수신자(=내 `client_id`) | 내 `client_id`와 일치하는지 확인 | 옆 동네 앱의 토큰을 내 앱에서 쓰는 [[Confused Deputy]] 공격 |
| `exp` | Expiration. 만료 Unix timestamp | 현재 시각이 `exp` 이전인지 확인 | 만료된 토큰을 영원히 수락 |
| `iat` | Issued At. 발급 Unix timestamp | 미래 시각으로 발급된 토큰 거부, 너무 오래된 토큰 거부 | 시계 왜곡/리플레이 보조 검증 불가 |
| `nonce` | 인가 요청 시 보낸 [[nonce]]가 그대로 반환됨 | `/authorize`에 보낸 nonce 값과 일치하는지 확인 | [[Replay Attack]] 차단 불가 (재사용된 ID Token을 못 잡음) |

> [!example] 사용자 식별은 `iss + sub` 조합으로
> `sub`는 *OP 안에서만* 고유하다. Google의 `sub=123`과 Apple의 `sub=123`은 다른 사람이다. 그래서 DB에 사용자 키를 잡을 때는 항상 **`(iss, sub)` 쌍**으로 묶어야 한다. 한 사용자가 구글 로그인과 애플 로그인을 둘 다 쓰면 레코드는 두 개여야 하고, 그걸 사람-한 명으로 합치는 건 별도의 계정 연결 로직이다.

> [!warning] `aud` 검증을 빼먹지 말 것
> 한 OP에 등록된 여러 앱이 같은 키로 ID Token을 받을 때, **`aud`를 검증하지 않으면 옆 앱이 받은 토큰을 내 앱이 그대로 수락**해 버린다. 서명도 진짜고 issuer도 같으니 다른 모든 검증을 통과한다. `aud === my_client_id` 한 줄을 반드시 넣어야 한다.

> [!tip] 검증 순서 권장
> ① 서명 검증 ([[JWKS]]에서 `kid`로 공개키 조회) → ② `iss` → ③ `aud` → ④ `exp`/`iat` → ⑤ `nonce`. 서명이 깨졌으면 클레임은 신뢰할 게 못 되니 가장 먼저 끝낸다. 대부분의 라이브러리([[jose]], [[jsonwebtoken]], [[python-jose]])가 이 순서를 자동으로 처리하지만, **`audience`와 `issuer`는 호출 시 명시적으로 넘겨야** 검증된다.

## JWT 서명 구조와 검증: 무엇으로 어떻게 서명하나

[[JWT]](JSON Web Token)의 안전성은 **암호화가 아니라 서명(Signature)** 에 달려 있다. 이 한 가지 사실을 오해하면 보안 전체가 무너진다.

> [!warning] JWT는 암호화되지 않는다
> Header와 Payload는 **단순 Base64URL 인코딩**일 뿐, 누구나 디코딩해 평문으로 읽을 수 있다. JWT가 보장하는 것은 **기밀성(Confidentiality)이 아니라 무결성(Integrity)과 진위성(Authenticity)** 이다. 따라서 패스워드·주민번호·결제정보 같은 민감 데이터를 Payload에 절대 넣지 말 것.

### 1. 3-part 구조: Header.Payload.Signature

JWT는 점(`.`)으로 구분된 세 덩어리로 구성된다.

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEyMyJ9 . eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJzdWIiOiIxMTAxNjkiLCJhdWQiOiJteS1jbGllbnQtaWQiLCJleHAiOjE3MTk4NDgwMDB9 . NHVB2tQs...XyZ
└────────── Header ──────────┘ └─────────────────── Payload ───────────────────┘ └ Signature ┘
```

| 부분 | 내용 | 예시 |
|---|---|---|
| **Header** | 알고리즘(`alg`), 키 ID(`kid`), 타입(`typ`) | `{"alg":"RS256","kid":"abc123","typ":"JWT"}` |
| **Payload** | Claims (`iss`, `sub`, `aud`, `exp`, `iat`, `nonce`, ...) | `{"iss":"https://accounts.google.com","sub":"110169","aud":"my-client-id","exp":1719848000}` |
| **Signature** | Header+Payload를 키로 서명한 결과 | 이진값을 base64url로 인코딩 |

### 2. 무엇을 대상으로 서명하나: `signing_input`

서명은 Header나 Payload 단독이 아니라 **둘을 점으로 이어붙인 문자열 전체**를 대상으로 한다.

```
signing_input = base64url(Header) + "." + base64url(Payload)
Signature     = Sign( signing_input, key )
```

이렇게 하면 Header의 `alg` 한 글자만 바꿔도, Payload의 `sub` 한 글자만 바꿔도 서명이 깨진다. **변조 = 즉시 탐지**.

> [!example] 서명 생성/검증 의사코드
> ```python
> # 서명 생성 (발급자)
> header_b64  = base64url(json.dumps({"alg":"RS256","kid":"abc123"}))
> payload_b64 = base64url(json.dumps({"iss":..., "sub":..., "exp":...}))
> signing_input = f"{header_b64}.{payload_b64}".encode()
> signature_b64 = base64url(RSA_Sign(signing_input, private_key))
> jwt = f"{header_b64}.{payload_b64}.{signature_b64}"
>
> # 검증 (Relying Party)
> h, p, s = jwt.split(".")
> signing_input = f"{h}.{p}".encode()
> assert RSA_Verify(signing_input, base64url_decode(s), public_key)  # 위변조 여부
> # 그 다음 Claims 검증 (iss / aud / exp / nonce ...)
> ```

### 3. 서명 방식 2갈래: 대칭 vs 비대칭

| 구분 | 대칭키 ([[HMAC]]) | 비대칭키 ([[RSA]] / [[ECDSA]]) |
|---|---|---|
| **대표 알고리즘** | `HS256`, `HS384`, `HS512` | `RS256`, `RS384`, `RS512`, `ES256`, `ES384`, `EdDSA` |
| **키 구조** | 단일 `secret` 하나 | `private_key`(개인키) + `public_key`(공개키) 쌍 |
| **서명** | secret으로 HMAC 계산 | 개인키로 서명 |
| **검증** | 같은 secret으로 HMAC 재계산 후 비교 | 공개키로 서명 검증 |
| **성능** | 매우 빠름 | 느림 (특히 RSA) |
| **공개 검증** | 불가능 (secret을 아는 자는 위조도 가능) | 가능 (공개키는 위조에 쓸 수 없음) |
| **권장 용도** | 단일 시스템 내 자체 세션 토큰 (발급=검증 같은 주체) | [[OIDC]], 분산 검증, 외부 클라이언트 |

> [!note] OIDC 표준은 비대칭 서명을 전제로 한다
> Google·Apple·[[Auth0]]·[[Keycloak]]·[[Okta]] 같은 OP는 **`RS256`(또는 `ES256`)** 으로 서명하고, 공개키만 [[JWKS]]로 전세계에 배포한다. **개인키는 OP만 보유**하므로 위조가 원천 차단되고, 검증자는 누구든 공개키로 무결성만 확인하면 된다. 발급자와 검증자가 분리돼도 신뢰가 성립하는 이유가 바로 이 비대칭성이다.

#### HS256 (대칭) 흐름

```
        ┌──────────┐    secret    ┌──────────┐
앱A ───►│ Sign HMAC├──────────────┤Verify HMAC│◄─── 앱A (자기 자신)
        └──────────┘   (공유)     └──────────┘
                  ⚠ secret을 아는 누구든 위조 가능
```

#### RS256/ES256 (비대칭) 흐름

```
   OP (Authorization Server)              Relying Party (Client/API)
  ┌────────────────────────┐             ┌──────────────────────────┐
  │  Private Key  ──Sign──►│  ID Token   │  Public Key (from JWKS)  │
  │  (오직 OP만)            │ ──────────► │  ──Verify──► OK / NG     │
  └────────────────────────┘             └──────────────────────────┘
        ✅ 개인키 없이는 유효한 서명 생성 불가
        ✅ 공개키는 검증만 가능, 위조 불가
```

### 4. RS256 검증 절차 (가장 흔한 케이스)

1. **JWT 디코드** → `header`, `payload`, `signature` 추출
2. **`alg` 화이트리스트 검증** → 우리가 허용한 `RS256`인지 (그리고 `none`이 아닌지)
3. **`kid` 추출** → Header의 `kid` 값으로 어느 공개키를 쓸지 결정
4. **JWKS에서 공개키 획득** → [[jwks_uri]]에서 캐싱된 키 조회, 없으면 다시 fetch (자세히는 [[JWKS]]·[[kid]] 섹션 참고)
5. **서명 검증** → `RSA-Verify(signing_input, signature, public_key)`
6. **Claims 검증** → `iss`(발급자), `aud`(=`client_id`), `exp`(만료), `nbf`, `iat`(skew 허용 범위), `nonce`(저장한 값과 일치)

> [!tip] 검증 순서는 위에서 아래로 엄격히
> 서명 검증이 실패하면 **Payload 안의 어떤 값도 신뢰하면 안 된다**. `iss`나 `aud`를 보고 어떤 키로 검증할지 결정하는 코드는 위험할 수 있다. 늘 **Header의 `kid`만으로 키를 고르고, 서명 통과 후에야 Claims를 본다.**

### 5. 공격 시나리오와 방어

#### ① 단순 변조 시도

공격자가 Payload의 `"role":"user"`를 `"role":"admin"`으로 바꾼다.

```
원본:     base64url(H) . base64url({"role":"user"})  . sig
변조:     base64url(H) . base64url({"role":"admin"}) . sig   ← signing_input이 바뀌었으니 sig가 더 이상 안 맞음
```

→ 서명 검증 단계에서 즉시 실패. **방어 끝**.

#### ② `alg: none` 공격

[[JWT]] 명세에는 `"alg":"none"`(서명 없음) 모드가 있었다. 일부 라이브러리가 이를 그대로 받아들이면, 공격자가 다음과 같이 보낼 수 있다.

```json
Header  : {"alg":"none"}
Payload : {"sub":"admin"}
Signature: (빈 문자열)
```

서버가 `alg`를 토큰에서 그대로 읽어 "아, 서명 안 했네, 통과!" 하면 끝.

> [!warning] alg는 헤더에서 읽지 말고 검증 측에서 못박아라
> 검증 코드는 **허용 알고리즘을 화이트리스트로 고정**해야 한다.
> ```python
> # ❌ 위험: 헤더의 alg를 그대로 신뢰
> jwt.decode(token, key, algorithms=[header["alg"]])
>
> # ✅ 안전: 우리가 기대하는 알고리즘만 허용
> jwt.decode(token, key, algorithms=["RS256"])  # "none" 자동 거부
> ```

#### ③ 알고리즘 혼동 공격 (RS256 → HS256)

서버가 RS256으로 발급된 토큰을 받는데, 검증 함수에 키 종류를 명시하지 않으면 공격자가:

1. Header의 `alg`를 `RS256` → **`HS256`** 으로 바꾼다.
2. 공개된 RSA **공개키 자체**를 HMAC의 **secret으로 사용**해 새 서명을 만든다.
3. 서버가 `header.alg`를 그대로 믿고 HS256으로 검증 → 같은 공개키를 secret으로 써서 HMAC 일치 → 통과.

```
공격자가 만든 토큰:
  Header  : {"alg":"HS256"}   ← RS256에서 변조
  Payload : {"sub":"victim"}
  Sig     : HMAC-SHA256(signing_input, RSA_public_key_bytes)
                                       └──── 누구나 다운로드 가능 ────┘
서버 (실수):
  알고리즘을 헤더 그대로 읽음 → HS256
  키는 "어차피 같은 키 자료" 라며 RSA 공개키 사용
  → HMAC 일치 → 통과 😱
```

방어는 동일하다: **`algorithms=["RS256"]` 로 못박기**, 그리고 키 객체에 알고리즘 타입을 강제하는 라이브러리 사용([[node-jsonwebtoken]], [[PyJWT]], [[jose]] 등은 옵션 제공).

#### ④ 키 혼동 / 키 누출

- 개인키가 유출되면 **모든 서명이 위조 가능**. → 즉시 [[Key Rotation]] (자세히는 [[JWKS]] 섹션).
- 환경별로(dev/staging/prod) 키를 분리하고, [[HSM]]·[[KMS]]에 보관.

### 6. 알고리즘 빠른 비교

| `alg` | 종류 | 키 길이/곡선 | 특징 |
|---|---|---|---|
| `HS256` | HMAC-SHA256 (대칭) | secret ≥ 256-bit | 빠르지만 단일 신뢰 도메인용 |
| `RS256` | RSA-PKCS1-v1_5 + SHA-256 | RSA 2048~4096 | OIDC 사실상 디폴트, 호환성 최강 |
| `RS384`, `RS512` | RS256 변형 | 동일 | 해시 길이 차이 |
| `ES256` | ECDSA + P-256 + SHA-256 | EC P-256 | 짧은 서명(~64B), 빠른 검증, 모바일 친화 |
| `ES384`, `ES512` | ECDSA P-384/P-521 | EC | 더 강한 보안 |
| `EdDSA` (Ed25519) | Edwards-curve | 256-bit | 가장 현대적, 결정론적 서명 |
| `PS256` | RSASSA-PSS + SHA-256 | RSA | 패딩이 더 안전 (랜덤 솔트) |
| `none` | 서명 없음 | — | **절대 금지** |

> [!tip] 새로 고른다면 ES256 또는 EdDSA
> RS256은 호환성 때문에 살아 있을 뿐, 새로 설계한다면 서명이 짧고 빠른 **ES256(P-256)** 이나 **EdDSA(Ed25519)** 가 일반적으로 더 좋은 선택이다.

### 7. 한 줄 정리

> [!note] 핵심 요약
> OIDC는 **개인키로 서명**하고 **JWKS의 공개키로 검증**한다. Header+Payload 전체를 대상으로 서명하므로 변조는 즉시 깨지고, 개인키 없이는 위조도 불가능하다. 보안의 마지막 한 줄은 **"`alg`는 헤더에서 읽지 말고 검증 측에서 못박아라."**

## Bearer Token 탈취 문제와 방어 전략

### 1. Bearer Token이란 무엇인가 — "소지=권한"의 함정

OAuth 2.0과 [[OIDC]]가 사용하는 토큰은 대부분 **Bearer Token(소지자 토큰)** 이다. 이름 그대로 "이 토큰을 *지니고 있는 자(bearer)* 가 곧 권한자"라는 뜻이다. 서버는 요청에 첨부된 토큰이 유효한지만 검사하고, "지금 이 요청을 보낸 사람이 *진짜* 토큰의 원 주인인가?"는 묻지 않는다.

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEyMyJ9.eyJzdWI...
```

> [!warning] 핵심 비유: Bearer Token은 **현금**이다
> 1만 원짜리 지폐는 누가 들고 있든 1만 원이다. 편의점 직원은 "이 지폐의 원래 주인이 누구인지" 묻지 않는다. Bearer Token도 똑같다. 누가 들고 와도 서버는 "유효한 토큰이네, OK" 하고 통과시킨다.

그래서 **탈취되면 곧바로 원 주인 행세가 가능**하다. 이는 토큰 기반 인증의 가장 근본적인 약점이며, 사용자의 질문 "토큰 탈취당하면 그 유저는 원 주인인척 행세할 수 있다는건데 이게 맞아?"의 답은 **명백히 그렇다(Yes)** 이다.

## 다이어그램

[[canvas/OIDC와_OAuth_2.0_완벽_이해.canvas|개념도]]

---

### 2. 서명이 있는데도 탈취를 못 막는 이유

JWT의 [[JWT Signature]]는 강력하다. 그런데 왜 탈취는 못 막을까?

| 서명이 막아주는 것 | 서명이 못 막는 것 |
|---|---|
| Payload 한 글자 변조 → 서명 깨짐 → 거부 | 토큰을 **통째로 복사**해서 그대로 사용 |
| 공격자가 가짜 토큰을 새로 만들기 → 개인키 없어 불가 | "지금 이 요청자가 누구인가" — 서명은 *내용*을 검증할 뿐, *소지자*는 검증 못 함 |
| `alg:none` 같은 알고리즘 우회 → 화이트리스트로 차단 | XSS·MITM·로그 유출 등으로 새어 나간 진짜 토큰 |

> [!note] 탈취된 Bearer Token은 **완벽하게 유효한 진짜 토큰**이다
> 서명은 멀쩡, `exp`도 멀쩡, `iss`/`aud`도 일치. 서버 입장에서 "정상 요청"과 "탈취 요청"은 **암호학적으로 구별 불가능**하다. 이게 핵심이다.

---

### 3. 다층 방어 철학 — "탈취를 0%로 만들 수는 없다"

토큰 탈취를 완전히 차단하는 단일 기술은 존재하지 않는다. 따라서 OIDC 보안은 **다층 방어(Defense in Depth)** 로 접근한다. 철학을 한 줄로 요약하면:

> [!tip] OIDC 보안 철학
> **"탈취 자체를 어렵게 + 탈취되어도 피해 최소화 + 빠른 탐지·무효화"**
> 어느 한 단계만 통과해서는 공격이 성공할 수 없도록 여러 겹의 방어벽을 쌓는다.

```
┌────────────────────────────────────────────────────────────┐
│  ① 탈취 방지       : HTTPS, HttpOnly Cookie, CSP, SameSite   │
│  ② 피해 최소화     : 짧은 Access Token TTL (5분~1시간)        │
│  ③ 소지=권한 깨기  : DPoP, mTLS (Sender-Constrained Token)   │
│  ④ 탐지·무효화     : Refresh Rotation, 블랙리스트, 이상탐지   │
└────────────────────────────────────────────────────────────┘
```

각 축을 하나씩 보자.

---

### 4. ① 짧은 수명 (Short-Lived Token)

가장 단순하고 가장 효과적인 방어다. **탈취돼도 곧 만료**되도록 한다.

| 토큰 종류 | 권장 수명 | 이유 |
|---|---|---|
| **Access Token** | 5분 ~ 1시간 | 매 요청에 노출. 새어 나갈 확률 높음. 짧게. |
| **ID Token** | 5분 ~ 1시간 | 보통 로그인 직후 1회 검증 후 폐기. |
| **Refresh Token** | 며칠 ~ 몇 주 | 안전 보관 가능. Rotation으로 보호 ([[Refresh Token Rotation]] 섹션 참고) |

> [!example] 수명이 짧으면 무엇이 좋은가
> Access Token TTL이 15분이라고 하자. 공격자가 [[XSS]]로 토큰을 훔쳐도 **최대 15분만 악용 가능**하다. 그 사이 사용자가 로그아웃하거나 토큰 재발급이 발생하면 더 빨리 무효화된다. 24시간짜리 토큰과 비교하면 공격 윈도우가 96배 줄어든다.

---

### 5. ② 전송·저장 보호 (Transport & Storage)

토큰이 새어 나갈 경로 자체를 막는다.

#### HTTPS 강제
- 평문 HTTP로 토큰을 보내면 와이파이 도청, [[MITM]] 공격으로 즉시 탈취된다.
- OIDC 명세는 모든 엔드포인트(`authorization_endpoint`, `token_endpoint`, `jwks_uri` 등)에 **HTTPS 강제**.

#### HttpOnly + Secure + SameSite Cookie
SPA에서 토큰을 어디 저장할지가 큰 이슈다. 옵션과 위험을 비교:

| 저장 위치 | XSS 시 탈취 | CSRF 위험 | 권장도 |
|---|---|---|---|
| `localStorage` | **즉시 탈취 가능** (JS로 읽힘) | 없음 | ✗ 지양 |
| `sessionStorage` | **즉시 탈취 가능** | 없음 | ✗ 지양 |
| 메모리(JS 변수) | 페이지에 XSS 코드 주입되면 탈취 | 없음 | △ Access Token만 |
| **HttpOnly Cookie** | **JS로 읽기 불가** | SameSite로 차단 | ✓ Refresh Token |

> [!tip] HttpOnly Cookie의 결정적 장점
> `document.cookie`로도 읽히지 않는다. 브라우저가 자동으로 요청에 첨부할 뿐, JavaScript는 그 값을 절대 볼 수 없다. [[XSS]]가 발생해도 토큰이 통째로 새어 나가지 않는다. `Secure`(HTTPS 전용) + `SameSite=Lax/Strict`([[CSRF]] 방지)를 함께 걸어야 완성된다.

```http
Set-Cookie: refresh_token=eyJhbGc...; HttpOnly; Secure; SameSite=Strict; Path=/auth
```

---

### 6. ③ 토큰 바인딩 — "소지=권한" 등식을 깨라

가장 근본적인 해결책이다. 토큰 자체에 **"이 토큰은 X 키를 가진 클라이언트만 쓸 수 있다"** 는 제약을 새겨 넣는다. 훔쳐도 키가 없으면 무용지물이 된다. 이런 토큰을 **Sender-Constrained Token**이라고 부른다.

#### DPoP (Demonstrating Proof-of-Possession) — RFC 9449

클라이언트가 키 쌍을 만들고, **매 API 요청마다 자기 개인키로 짧은 증명 JWT(DPoP Proof)** 를 동봉한다.

```http
POST /api/resource
Authorization: DPoP eyJhbGc...           ← Access Token
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIs...        ← 클라이언트가 매번 새로 서명한 증명
```

- 서버는 두 가지를 검증:
  1. Access Token이 유효한가
  2. DPoP Proof가 그 토큰에 바인딩된 **공개키 지문(`jkt`)** 으로 서명됐는가
- 공격자가 Access Token만 훔치고 개인키를 못 훔치면 → DPoP Proof를 만들 수 없음 → **거부**.

#### mTLS (Mutual TLS) — RFC 8705

TLS 핸드셰이크 단계에서 **클라이언트도 인증서를 제시**한다. Access Token에는 그 인증서의 지문(`x5t#S256`)이 박혀 있다. 다른 인증서로 같은 토큰을 쓰면 거부.

> [!warning] DPoP / mTLS는 강력하지만 비용이 있다
> 구현 복잡도, 키 관리, 클라이언트 호환성 부담이 크다. 그래서 **금융·고가치 API**(예: PSD2 Open Banking)에 주로 쓰이고, 일반 웹앱은 ①②④로 막는 게 보통이다. 하지만 "탈취돼도 쓸 수 없게" 만드는 가장 확실한 답은 이쪽이다.

---

### 7. ④ 탐지 & 무효화

탈취가 일어났을 때 빠르게 알아채고 끊어내는 축이다.

- **[[Refresh Token Rotation]] + Reuse Detection**: Refresh Token을 일회용으로 만들고, 폐기된 토큰이 다시 등장하면 탈취로 판정해 token family 전체를 무효화. (자세한 건 별도 섹션)
- **이상 탐지(Anomaly Detection)**: 같은 토큰이 서울/뉴욕에서 동시 사용, User-Agent 급변, IP 대역 점프 등 → 강제 재인증.
- **블랙리스트(Revocation List)**: 로그아웃·비밀번호 변경·탈취 의심 시 토큰 jti를 [[Redis]] 등에 넣고, 모든 요청 시 조회.
- **Step-Up Authentication**: 민감 작업(송금, 계정 삭제)은 토큰이 있어도 추가 [[MFA]] 요구.

#### JWT 무효화의 딜레마

> [!warning] JWT의 양날의 검: 자체 완결성
> JWT는 서명만으로 검증되어 **서버가 매 요청에 DB를 조회하지 않아도 된다**(성능 장점). 하지만 그 말은 서버가 "이 토큰은 무효" 라고 *직접* 알릴 수단이 없다는 뜻이다. `exp`까지 기다리거나, 블랙리스트를 두는 순간 매 요청 [[Redis]] 조회가 부활해 JWT의 장점이 일부 상쇄된다.
>
> 그래서 현실의 해법은: **Access Token은 짧게(블랙리스트 안 쓰고 만료 대기) + Refresh Token만 블랙리스트(빈도 낮음)**. 짧은 수명이 곧 무효화 비용을 낮춰주는 것이다.

---

### 8. 종합 — 한 장 정리

```
탈취 시나리오             ↓ 방어 단계
─────────────────────────────────────────────────────
도청 (와이파이/MITM)   →  ① HTTPS
XSS로 토큰 탈취        →  ② HttpOnly Cookie + CSP
                          ②' Refresh Token 분리(메모리 vs Cookie)
탈취 후 장기 악용      →  ③ 짧은 Access Token TTL
훔쳐도 못 쓰게         →  ④ DPoP / mTLS (토큰 바인딩)
Refresh Token 탈취     →  ⑤ Rotation + Reuse Detection
                          ⑥ Token Family 일괄 폐기
탈취 탐지              →  ⑦ 이상 탐지 + 블랙리스트
민감 작업              →  ⑧ Step-Up MFA
```

> [!tip] 한 줄 결론
> Bearer Token은 **소지=권한**이라는 단순함이 강점이자 약점이다. 어느 한 기술도 완벽하지 않으므로, **짧게 + 안전하게 보관 + 바인딩 + 빠른 무효화**의 네 축을 동시에 쌓아 *탈취 확률 × 악용 시간 × 피해 범위*를 모두 줄이는 것이 OIDC 보안의 핵심 철학이다.

## Refresh Token Rotation과 Reuse Detection

[[Refresh Token]] Rotation은 "한 번 쓴 Refresh Token은 곧바로 폐기하고 새 토큰으로 교체한다"는 단순한 규칙 하나로, 장수명 토큰 탈취라는 [[Bearer Token]]의 가장 큰 약점을 능동적으로 탐지·차단하도록 만든 방어 기법이다. 핵심은 *일회용화 + 폐기 이력 추적 + family 단위 무효화* 세 가지의 조합이다.

### 왜 Refresh Token이 가장 위험한 자산인가

[[Access Token]]은 보통 5분~1시간으로 짧지만, [[Refresh Token]]은 며칠~수십일 단위로 길다. 즉 Refresh Token이 탈취되면 공격자는 만료일까지 *원하는 만큼 새 Access Token을 찍어낼 수* 있다. Access Token을 잃는 것보다 훨씬 큰 사고다.

> [!warning] Refresh Token은 "Access Token 발급기"
> Access Token 탈취 = 30분짜리 사고. Refresh Token 탈취 = 며칠~몇 주에 걸친 지속적 접근. 그래서 Refresh Token에는 Access Token보다 훨씬 강한 보호가 필요하다.

### Rotation이 없을 때: 고정 Refresh Token의 함정

```
[로그인] → RT_AAA 발급 (유효기간 30일)
  /token (RT_AAA) → 새 AT + 같은 RT_AAA   ← 계속 재사용
  /token (RT_AAA) → 새 AT + 같은 RT_AAA
  /token (RT_AAA) → 새 AT + 같은 RT_AAA
  ...
```

공격자가 어느 시점에 `RT_AAA`를 훔치면:

- 만료일까지 무한히 새 Access Token 발급 가능
- 정상 사용자도 같은 RT_AAA를 계속 쓰므로, 서버 입장에서 **"누가 진짜 주인인지" 구분할 단서가 없다**
- 탐지 자체가 불가능 → 사용자가 "내 계정이 이상해요"라고 신고하기 전까지 침입을 모름

### Rotation이 있을 때: 일회용 Refresh Token

매 갱신마다 새 RT를 발급하고, 직전 RT는 즉시 폐기한다.

```
[로그인] → RT_AAA  (state=ACTIVE)
  /token (RT_AAA) → 새 AT + RT_BBB
                    ↳ RT_AAA: state=ROTATED (폐기, 더 못 씀)
  /token (RT_BBB) → 새 AT + RT_CCC
                    ↳ RT_BBB: state=ROTATED
  /token (RT_CCC) → 새 AT + RT_DDD
                    ↳ RT_CCC: state=ROTATED
  ...
```

이제 *각 Refresh Token은 정확히 한 번만 유효*하다. 탈취되더라도 한 번 쓰면 끝나는 토큰이 된다.

### Reuse Detection: 폐기된 RT가 다시 등장하면?

Rotation의 진짜 위력은 "한 번 더 쓰일 때" 발현된다.

> [!example] 탈취 시나리오 — 두 명이 같은 체인을 쓰면
> 1. 사용자가 정상 사용 중: `RT_BBB`를 들고 있음
> 2. 공격자가 어떤 경로(XSS, MITM, 백업 유출 등)로 `RT_BBB`를 훔침
> 3. 공격자가 먼저 `/token (RT_BBB)` 호출 → 새 AT + `RT_CCC` 수령. 서버는 `RT_BBB`를 ROTATED로 표시
> 4. 시간이 흘러 사용자의 Access Token이 만료됨 → 사용자 클라이언트가 자기가 들고 있던 `RT_BBB`로 갱신 시도
> 5. 서버: "어? `RT_BBB`는 이미 ROTATED 상태인데 또 쓰이네?" ← **Reuse Detection 발동**

폐기된 Refresh Token이 다시 등장한다는 것은, 같은 체인을 두 주체가 들고 있다는 명백한 증거다. 정상 흐름에서는 절대 일어날 수 없는 사건이다.

### Token Family: family 단위로 무효화한다

각 로그인 세션에서 파생된 Refresh Token들은 하나의 **Token Family**로 묶인다. Reuse가 감지되면 그 family에 속한 *모든* 토큰을 한꺼번에 폐기한다.

```
Login Session #42 → Family ID: fam_42
  ├── RT_AAA (ROTATED)
  ├── RT_BBB (ROTATED) ← 여기서 reuse 시도 감지!
  ├── RT_CCC (ACTIVE)  ← 공격자 손에 있을 수도, 사용자 손에 있을 수도
  └── (이후 발급될 모든 토큰)

→ fam_42 전체를 REVOKED 처리
→ 공격자/사용자 모두 재로그인 강제
```

> [!note] 왜 family 전체를 죽이나
> Reuse가 감지된 시점에 *어느 쪽이 공격자인지 알 수 없다*. 가장 안전한 가정은 "둘 중 하나는 적이다"이고, 그러면 그 체인에서 파생된 모든 토큰을 못 믿는 것이 정답이다. 사용자에게는 재로그인이라는 작은 불편을, 공격자에게는 완전한 차단이라는 결과를 준다.

### 서버에 무엇을 저장해야 하나

[[JWT]]는 자체완결적(self-contained)이라 서버 상태가 필요 없지만, Rotation과 Reuse Detection은 *서버 측 상태*를 반드시 요구한다. 보통 [[Redis]]나 RDB에 다음과 같이 둔다.

| 컬럼 | 의미 |
|------|------|
| `jti` | 이 Refresh Token의 고유 ID (JWT의 jti 클레임) |
| `family_id` | 이 토큰이 속한 Token Family |
| `user_id` | 어떤 사용자에게 발급됐는지 |
| `parent_jti` | 어느 RT를 갱신해서 만들어졌는지 (체인 추적용) |
| `state` | `ACTIVE` / `ROTATED` / `REVOKED` |
| `issued_at`, `expires_at` | 발급/만료 시각 |
| `replaced_by_jti` | rotation으로 이어 만든 다음 RT의 jti |

검증 로직 의사코드:

```python
def exchange_refresh_token(rt):
    record = store.get(rt.jti)

    if record is None:
        raise InvalidToken()

    if record.state == "ROTATED":
        # 이미 한 번 쓰인 RT가 다시 왔다 → 탈취 의심
        store.revoke_family(record.family_id)
        audit_log("RT reuse detected", family=record.family_id)
        raise SecurityIncident()

    if record.state == "REVOKED":
        raise RevokedToken()

    # 정상 흐름
    new_at = issue_access_token(record.user_id)
    new_rt = issue_refresh_token(
        user_id=record.user_id,
        family_id=record.family_id,
        parent_jti=record.jti,
    )
    record.state = "ROTATED"
    record.replaced_by_jti = new_rt.jti
    store.save(record)
    return new_at, new_rt
```

> [!tip] 멱등성과 네트워크 재시도 함정
> 클라이언트가 `/token`을 보냈는데 응답이 끊겨 같은 RT로 한 번 더 재시도하면, 정상 흐름인데도 Reuse로 오인될 수 있다. 실무에서는 짧은 grace window(예: 같은 RT의 마지막 사용 후 N초 내 재호출은 같은 결과 반환)를 두거나, idempotency key를 받거나, 클라이언트가 "갱신 중" 상태에서 동시 호출을 막는 방식으로 완화한다.

### 표준 근거: [[OAuth 2.1]] / RFC 6749 Security BCP

Refresh Token Rotation과 Sender-Constrained / Reuse Detection은 [[OAuth 2.1]] 초안과 [OAuth 2.0 Security Best Current Practice (draft-ietf-oauth-security-topics)](https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics/)에서 **공개 클라이언트(SPA, 모바일 앱)에서 사실상 의무**로 명시된다. 즉 "선택 사항"이 아니라 "공개 클라이언트라면 이렇게 해야 한다"는 표준 권장 사항.

### 다른 방어 축과의 관계

| 축 | 막는 것 | 한계 |
|----|---------|------|
| 짧은 Access Token 수명 | AT 탈취 피해 시간 제한 | RT 탈취는 못 막음 |
| HttpOnly Cookie / HTTPS | 탈취 자체 차단 | XSS 우회/내부 유출은 막지 못함 |
| **Rotation + Reuse Detection** | **RT 탈취를 사후 탐지·차단** | 첫 번째 오용까지는 못 막음 |
| [[DPoP]] / [[mTLS]] | 키 바인딩으로 토큰만으로는 쓰지 못하게 | 구현 복잡 |

Rotation은 *예방*이 아니라 *탐지와 봉쇄*다. 다른 축들과 다층 방어로 함께 써야 의미가 있다.

> [!note] 한 줄 요약
> Refresh Token Rotation = "RT를 일회용으로 만들고, 폐기된 RT가 다시 쓰이는 순간을 탈취 신호로 삼아 그 family 전체를 즉시 무효화하는" 능동적 탐지 기반 방어 기법.

## JWKS와 kid: 공개키 배포와 키 회전

OIDC 토큰 검증의 마지막 퍼즐 조각은 "그래서 OP의 공개키를 **어디서**, **어떻게** 가져오나"이다. 이 문제를 표준화한 것이 [[JWKS]]이고, 여러 키 중에 올바른 키를 고르게 해주는 식별자가 [[kid]]다.

### 1. JWKS란 무엇인가 — 공개키 카탈로그

**JWKS (JSON Web Key Set)** = "서명 검증에 쓸 공개키들을 JSON 형식으로 모아 공개해 놓은 목록"이다.

비대칭 서명([[RS256]] / [[ES256]])에서 [[OpenID Provider]]는 **개인키로 서명**하고, [[Relying Party]](앱)은 **공개키로 검증**한다. 그렇다면 앱은 그 공개키를 어디서 구할까?

- ❌ 코드에 하드코딩 → OP가 키를 교체하는 순간 전세계 앱이 깨진다.
- ❌ 수동 설정 파일 → 키 회전 때마다 운영자가 갱신해야 함.
- ✅ OP가 **표준 URL에 JSON으로 공개**, 앱이 **HTTP로 가져와 캐싱** → JWKS.

```
                 ┌─────────────────────────────────┐
   Google OP     │  https://www.googleapis.com/    │
   (서명자)       │     oauth2/v3/certs             │  ← jwks_uri
                 │  { "keys": [ {kid:..., n:..., e:...}, ... ] }
                 └─────────────────────────────────┘
                          ▲ 누구나 HTTP GET 가능
                          │ (공개키니까 비밀 X)
                          │
   My App ──── fetch ─────┘
   (검증자)
```

> [!note] JWK vs JWKS
> - **JWK** (JSON Web Key) = 키 **1개**를 JSON으로 표현한 것.
> - **JWKS** (JSON Web Key **Set**) = JWK들의 **배열**. `keys` 필드 하나로 묶여 있음.

### 2. JWKS 실제 모양

[[Google]] OP의 JWKS를 가져오면 대략 이런 JSON이 나온다:

```json
{
  "keys": [
    {
      "kid": "abc123def456",
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78L...",
      "e": "AQAB"
    },
    {
      "kid": "xyz789ghi012",
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "n": "qjQ5UfdNZQYrkP6QSj3FjUL7L8DDcXkR9k...",
      "e": "AQAB"
    }
  ]
}
```

각 필드의 의미:

| 필드 | 의미 |
|------|------|
| `kid` | **Key ID** — 이 키의 식별자. 토큰 헤더의 `kid`와 매칭에 사용 |
| `kty` | Key Type — `RSA`, `EC` 등 키 종류 |
| `use` | 용도 — `sig`(서명 검증) / `enc`(암호화). OIDC는 `sig` |
| `alg` | 이 키와 함께 사용해야 할 서명 알고리즘 (`RS256` 등) |
| `n` | RSA modulus (Base64URL) — 공개키의 핵심 |
| `e` | RSA exponent (보통 `AQAB` = 65537) |

> [!tip] n과 e의 정체
> RSA 공개키는 수학적으로 `(n, e)` 쌍이다. JWKS는 PEM이나 X.509 같은 별도 포맷이 아니라 이 두 숫자를 **그대로 JSON에 박아 넣는다**. 라이브러리가 이걸 받아 내부적으로 RSA Public Key 객체를 복원해서 검증한다.

### 3. kid — 여러 키 중 "이 토큰을 서명한 키"를 가리키는 이름표

[[OpenID Provider]]는 일반적으로 **공개키를 여러 개 동시에 게시**한다. 왜냐하면 **키 회전(rotation)** 때문이다 — 새 키와 옛 키가 한동안 공존한다.

그러면 앱은 어떤 키로 검증해야 할지 어떻게 알까? → 토큰의 **Header에 박혀있는 `kid`로 매칭**한다.

```
┌─────────────────────────────────────────────────────────────┐
│  ID Token (JWT)                                             │
│                                                             │
│  Header (decoded):                                          │
│  ┌────────────────────────────────────┐                     │
│  │ {                                  │                     │
│  │   "alg": "RS256",                  │                     │
│  │   "kid": "abc123def456" ───────────┼──┐                  │
│  │ }                                  │  │                  │
│  └────────────────────────────────────┘  │                  │
└──────────────────────────────────────────┼──────────────────┘
                                           │ 매칭
                                           ▼
┌─────────────────────────────────────────────────────────────┐
│  JWKS                                                       │
│  {                                                          │
│    "keys": [                                                │
│      { "kid": "abc123def456", "n": "...", "e": "AQAB" },  ←✓│
│      { "kid": "xyz789ghi012", "n": "...", "e": "AQAB" }    │
│    ]                                                        │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘
```

> [!example] kid 매칭 의사코드
> ```python
> header = decode_header(id_token)
> kid    = header["kid"]           # "abc123def456"
> jwks   = fetch_jwks(jwks_uri)    # 캐시 우선
> 
> key = next(k for k in jwks["keys"] if k["kid"] == kid)
> if not key:                       # 모르는 kid → 키 회전 가능성
>     jwks = fetch_jwks(jwks_uri, force_refresh=True)
>     key  = next(k for k in jwks["keys"] if k["kid"] == kid)
> 
> public_key = build_rsa_key(key["n"], key["e"])
> verify_signature(id_token, public_key, alg=key["alg"])
> ```

### 4. Key Rotation — 코드 수정 없이 자동 대응

**키 회전 시나리오**: Google이 보안 정책상 6개월마다 서명용 개인키를 새것으로 바꾼다고 하자.

#### Before (하드코딩 방식)

```
Day 1   : 앱 코드에 PUBLIC_KEY_v1 박혀있음 → 잘 검증됨
Day 180 : Google이 키 교체 → 새 토큰은 PRIVATE_KEY_v2로 서명
Day 181 : 앱이 새 토큰 검증 → PUBLIC_KEY_v1로 확인 → ❌ FAIL
          → 전세계 사용자 로그인 깨짐 → 긴급 배포
```

#### After (JWKS + kid 방식)

```
Day 1   : JWKS = [ {kid: "v1", ...} ]
          토큰 header.kid = "v1"
          앱이 JWKS에서 "v1" 찾아 검증 ✓

Day 175 : Google이 새 키 추가 (Overlap 기간)
          JWKS = [ {kid: "v1", ...}, {kid: "v2", ...} ]
          기존 토큰들 (kid: v1)은 여전히 검증 가능 ✓

Day 180 : Google이 새 토큰부터 v2로 서명 시작
          앱이 받은 토큰 header.kid = "v2"
          → 캐시에 v2 없음 → JWKS 다시 fetch → v2로 검증 ✓

Day 200 : Google이 v1 키 JWKS에서 제거
          (이미 v1으로 서명된 토큰은 모두 만료됨)
          JWKS = [ {kid: "v2", ...} ]
```

> [!note] 회전이 무중단인 이유
> 새 키를 **즉시 단독 교체**가 아니라 **옛 키와 함께 게시**한 뒤 천천히 옛 키를 빼는 **Overlap 전략**. 그래서 앱은 코드 수정 없이도 자연스럽게 새 키로 넘어간다. 이 모든 게 가능한 이유는 토큰이 자기 kid를 들고 다니기 때문이다.

### 5. 앱의 캐싱 전략 — JWKS는 어떻게 가져와야 하는가

JWKS를 **매 요청마다 fetch**하면 어떻게 될까?

- API 호출마다 OP에 HTTP 요청 → **OP에 과부하**, 앱 응답 속도 저하
- 네트워크 장애 시 인증 자체가 마비
- Google 같은 곳은 rate limit으로 차단 가능

그래서 **캐싱이 필수**다. 다만 단순한 캐싱은 키 회전 시 새 kid를 못 찾는다. 그래서 보통 이렇게 한다:

| 전략 | 동작 |
|------|------|
| **TTL 기반 캐싱** | JWKS를 10분~1시간 캐시. `Cache-Control` 헤더 존중 |
| **kid miss 시 강제 갱신** | 토큰 kid가 캐시에 없으면 즉시 JWKS 재요청 |
| **Rate limit** | kid miss가 발생해도 OP에 폭주하지 않게 (예: 10분에 1회만 강제 갱신) |
| **Background refresh** | TTL 만료 전 미리 백그라운드에서 갱신 (옵션) |

> [!warning] kid miss 시 무한 fetch 주의
> 공격자가 의도적으로 **존재하지 않는 kid**를 가진 토큰을 보내면, 순진한 구현은 매번 JWKS를 다시 가져온다 → OP에 대한 **증폭 공격**이 된다. 라이브러리는 보통 `jwksRequestsPerMinute` 같은 옵션으로 fetch 빈도에 상한을 둔다.

### 6. 라이브러리 동작 — `jwks-rsa`를 예로

[[Node.js]]에서 가장 많이 쓰는 [[jwks-rsa]] 라이브러리는 위 전략을 거의 그대로 구현한다:

```javascript
import jwksClient from 'jwks-rsa';
import jwt from 'jsonwebtoken';

const client = jwksClient({
  jwksUri: 'https://www.googleapis.com/oauth2/v3/certs',
  cache: true,                  // 메모리 캐싱
  cacheMaxEntries: 5,           // 최대 5개 키 캐싱
  cacheMaxAge: 10 * 60 * 1000,  // 10분 TTL
  rateLimit: true,
  jwksRequestsPerMinute: 10,    // 분당 최대 10회 fetch
});

function getKey(header, callback) {
  // header.kid를 받아 알맞은 공개키를 라이브러리가 알아서 가져옴
  client.getSigningKey(header.kid, (err, key) => {
    callback(err, key.getPublicKey());
  });
}

jwt.verify(idToken, getKey, {
  algorithms: ['RS256'],        // ← alg 화이트리스트로 alg:none, HS256 혼동 차단
  issuer: 'https://accounts.google.com',
  audience: MY_CLIENT_ID,
}, (err, payload) => {
  if (err) return reject(err);
  resolve(payload);
});
```

> [!example] 동작 흐름
> 1. `jwt.verify`가 토큰의 header에서 `kid` 추출
> 2. `getKey` 콜백에 header를 넘김
> 3. `jwks-rsa`가 캐시에서 해당 `kid`의 공개키 검색
> 4. 캐시 미스면 `jwksUri`로 HTTP GET → JWKS 파싱 → 캐시 갱신
> 5. 매칭된 키의 `n`, `e`로 RSA Public Key 객체 빌드
> 6. `jsonwebtoken`이 그 키로 서명 검증 + iss/aud/exp 검증

[[Python]]에서는 `PyJWT` + `PyJWKClient`, [[Java]]에서는 `nimbus-jose-jwt`의 `RemoteJWKSet`, [[Go]]에서는 `lestrrat-go/jwx`의 `jwk.Cache`가 같은 일을 한다.

### 7. 한 줄 정리

> [!tip] JWKS와 kid 한 줄 요약
> **JWKS**는 OP가 공개한 **서명 검증용 공개키 카탈로그**이고, **kid**는 토큰과 카탈로그 속 키를 **이름표로 매칭**시켜 주는 식별자다. 둘이 결합되어 OP는 자유롭게 키를 회전하고, 앱은 코드 수정 없이 자동으로 따라간다 — 비유하자면 **JWKS = 열쇠보관함, kid = 열쇠 이름표**.

## OIDC Discovery: /.well-known/openid-configuration

[[OIDC Discovery]]는 **"OP(OpenID Provider)가 자기 자신에 대한 모든 메타데이터를 표준 URL에 JSON으로 공개해 두는 메커니즘"**이다. 클라이언트(RP)는 `issuer` 하나만 알면 거기에 매달려 있는 엔드포인트·키 위치·지원 알고리즘을 자동으로 끌어와 OIDC 연동을 구성할 수 있다. 즉, **하드코딩의 종말**이자 OP가 직접 운영하는 "사용설명서"다.

### 왜 필요한가 — 하드코딩의 비극

[[OIDC]]를 수동으로 연동하려면 앱이 알아야 하는 것들이 줄줄이 있다.

```
authorization_endpoint  →  로그인 화면을 어디로 띄울지
token_endpoint          →  code를 토큰으로 어디서 교환할지
userinfo_endpoint       →  추가 프로필을 어디서 받을지
jwks_uri                →  서명 검증용 공개키를 어디서 받을지
issuer                  →  ID Token의 iss 클레임이 무엇과 일치해야 하는지
지원 알고리즘            →  RS256? ES256? PKCE method? 어떤 scope?
```

이걸 각 OP(Google, Apple, Auth0, Keycloak, Okta…)마다 코드에 박아두면:

- OP가 엔드포인트 URL을 살짝만 바꿔도 전 세계 RP가 깨진다.
- 키 회전(rotation), 알고리즘 추가 같은 일상적 운영 변경에도 매번 코드 배포가 필요하다.
- 오타 한 글자가 곧 사일런트 장애다.

> [!note] Discovery의 본질
> Discovery는 "**OP가 자기 자신을 어떻게 부르고 어떻게 사용해야 하는지**"를 표준 JSON으로 자기 입으로 말해 두는 것이다. RP는 듣기만 하면 된다.

### 위치: /.well-known/openid-configuration

표준 경로는 [[RFC 5785]]가 정의한 `.well-known/`을 따른다. OIDC는 그 아래 `openid-configuration` 을 강제 표준으로 지정했다 ([[OIDC Discovery 1.0 spec]]).

```
https://{issuer}/.well-known/openid-configuration
```

실제 예시:

| OP        | Discovery URL                                                          |
|-----------|------------------------------------------------------------------------|
| Google    | `https://accounts.google.com/.well-known/openid-configuration`         |
| Microsoft | `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration` |
| Auth0     | `https://{your-tenant}.auth0.com/.well-known/openid-configuration`     |
| Keycloak  | `https://{host}/realms/{realm}/.well-known/openid-configuration`       |

> [!tip] issuer 하나로 시작한다
> 사용자가 RP에게 알려줘야 하는 정보는 `issuer` (혹은 그것을 추출할 수 있는 도메인) 단 하나다. 거기에 `/.well-known/openid-configuration`을 붙이는 순간 나머지 전부가 따라온다.

### 응답 구조 — 실전 JSON

Google의 실제 응답에서 핵심만 추리면 다음과 같다.

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "revocation_endpoint": "https://oauth2.googleapis.com/revoke",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",

  "response_types_supported": ["code", "token", "id_token", "code token", "code id_token", "..."],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "email", "profile"],
  "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic"],
  "claims_supported": ["aud", "email", "email_verified", "exp", "family_name", "given_name", "iat", "iss", "locale", "name", "picture", "sub"],
  "code_challenge_methods_supported": ["plain", "S256"],
  "grant_types_supported": ["authorization_code", "refresh_token", "..."]
}
```

각 필드의 의미:

| 필드                                          | 의미                                                                                    |
|----------------------------------------------|-----------------------------------------------------------------------------------------|
| `issuer`                                     | OP의 고유 식별자. [[ID Token]]의 `iss` 클레임은 **반드시 이 값과 정확히 일치**해야 한다. |
| `authorization_endpoint`                     | 사용자를 로그인/동의 화면으로 보내는 URL ([[Authorization Code Flow]]의 시작점)         |
| `token_endpoint`                             | `code → access_token + id_token` 교환을 수행하는 URL                                    |
| `userinfo_endpoint`                          | Access Token으로 호출해 추가 프로필 클레임을 받는 URL                                   |
| `jwks_uri`                                   | [[JWKS]] (서명 검증용 공개키들)이 살고 있는 URL                                         |
| `revocation_endpoint`                        | 토큰 무효화(폐기) 엔드포인트                                                            |
| `id_token_signing_alg_values_supported`      | ID Token이 서명될 때 사용 가능한 알고리즘 (`RS256`, `ES256` 등 — 화이트리스트의 근거)   |
| `scopes_supported` / `claims_supported`      | 요청 가능한 scope, 발급되는 claim 목록                                                  |
| `code_challenge_methods_supported`           | [[PKCE]] 챌린지 방식 (`S256` 권장)                                                      |
| `token_endpoint_auth_methods_supported`      | 토큰 엔드포인트에 클라이언트가 자기 자신을 인증하는 방식                                |
| `response_types_supported` / `grant_types_supported` | OP가 지원하는 flow 종류                                                          |

> [!warning] issuer 검증은 "문자열 정확 일치"
> Discovery 문서의 `issuer` 값과, 받은 [[ID Token]]의 `iss` 클레임은 **문자 단위로 정확히** 같아야 한다. 트레일링 슬래시(`/`) 하나 차이도 곧 검증 실패다. 다중 테넌트 OP는 `issuer`에 테넌트 식별자가 포함되므로 더더욱 엄격하게 비교한다.

### Discovery → JWKS → kid → 검증, 전체 그림

Discovery는 그 자체로 끝이 아니라 **"전체 OIDC 검증 파이프라인의 시작점"**이다. 한 번의 로그인이 어떻게 이 메타데이터로부터 풀려나가는지 보자.

```
┌──────────────────────────────────────────────────────────────────────┐
│  부트스트랩: 앱 시작 또는 첫 요청 시 1회                                │
│                                                                      │
│  1. issuer ("https://accounts.google.com") 를 안다                    │
│  2. GET https://accounts.google.com/.well-known/openid-configuration │
│     → authorization_endpoint, token_endpoint, jwks_uri, ...          │
│  3. GET {jwks_uri}                                                   │
│     → JWKS = { keys: [ {kid: "abc", n, e, ...}, {kid: "xyz", ...} ] }│
│  4. Discovery 문서와 JWKS를 메모리에 캐싱 (TTL은 Cache-Control 존중)  │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  로그인 흐름: Authorization Code + PKCE                                │
│                                                                      │
│  5. 사용자를 authorization_endpoint 로 리다이렉트                       │
│     (client_id, redirect_uri, scope=openid, state, nonce, PKCE)      │
│  6. OP가 redirect_uri 로 ?code=... 반환                                │
│  7. POST token_endpoint 로 code 교환                                  │
│     → { access_token, id_token, refresh_token? }                     │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ID Token 검증                                                        │
│                                                                      │
│  8. id_token.header → { alg: "RS256", kid: "abc" }                    │
│  9. 캐시된 JWKS에서 kid="abc" 인 키 선택                                │
│       (없으면? → JWKS를 다시 fetch → 그래도 없으면 검증 실패)            │
│ 10. RSA-Verify(signing_input, signature, 그 공개키)                    │
│ 11. Claims 검증:                                                       │
│       iss === Discovery.issuer                                       │
│       aud === client_id                                              │
│       exp > now                                                      │
│       nonce === (저장해 둔 nonce)                                      │
│       alg ∈ id_token_signing_alg_values_supported 화이트리스트         │
│ 12. 모두 통과 → 로그인 성공                                            │
└──────────────────────────────────────────────────────────────────────┘
```

> [!example] 비유로 한 번에
> - [[Discovery]] = OP의 **사용설명서 / 안내데스크**
> - `jwks_uri` = **열쇠보관함의 주소**
> - [[JWKS]] = **열쇠보관함** 자체
> - [[kid]] = 열쇠에 달린 **이름표**
> - [[ID Token]] header.kid = "abc123 열쇠로 잠갔어요"라는 **꼬리표**
>
> 앱은 안내데스크에 가서 열쇠보관함 주소를 듣고 → 보관함을 열고 → 꼬리표가 가리키는 열쇠를 꺼내 → 자물쇠(서명)를 확인한다.

### 실전 라이브러리 — 손으로 구현하지 말 것

대부분의 언어에 [[OIDC]] 클라이언트 라이브러리가 있고 Discovery를 자동으로 처리한다. 그래서 실제 코드는 거의 항상 이 정도다.

```ts
// Node.js (openid-client) 예시
import { Issuer } from 'openid-client'

const issuer = await Issuer.discover('https://accounts.google.com')
// → 내부에서 GET /.well-known/openid-configuration 한 뒤
//   issuer.metadata 에 모든 엔드포인트가 채워짐

const client = new issuer.Client({
  client_id: process.env.CLIENT_ID,
  client_secret: process.env.CLIENT_SECRET,
  redirect_uris: ['https://app.example.com/callback'],
  response_types: ['code'],
})

// 로그인 콜백에서:
const tokens = await client.callback(redirect_uri, params, { nonce, state, code_verifier })
//   → 토큰 교환 + ID Token 서명/iss/aud/exp/nonce 검증까지 자동
//   → 내부적으로 jwks_uri 도 자동으로 fetch & cache & rotation 대응
```

[[jwks-rsa]] 같은 보조 라이브러리는 `jwks_uri`만 받으면 캐싱·키 회전·rate limit까지 알아서 처리해 준다.

### 캐싱·키 회전·장애 모드

> [!warning] 모든 요청마다 Discovery/JWKS를 부르지 말 것
> Discovery 문서와 JWKS는 거의 변하지 않으며, 호출량이 폭발하면 OP가 rate limit을 걸거나 차단한다. **메모리에 캐싱**하되 `Cache-Control: max-age`를 존중하고, **모르는 `kid`가 들어오면 그때만** JWKS를 강제로 재조회하는 게 정석이다.

전형적 캐시 전략:

| 자원              | 캐시 TTL                | 무효화 트리거                                       |
|------------------|------------------------|-----------------------------------------------------|
| Discovery 문서   | 수 시간 ~ 24시간       | TTL 만료, 명시적 재시작                              |
| JWKS             | 수 시간                | 토큰 헤더의 `kid`가 캐시에 없을 때 즉시 강제 재조회 |
| 부정적 캐시(없는 kid) | 짧게 (수 분)      | DoS 방지용 — 모르는 kid를 받을 때마다 fetch 폭주를 막기 위함 |

### 보안 체크리스트

> [!warning] Discovery를 신뢰하기 전에
> - **HTTPS 강제**: `.well-known/openid-configuration`은 반드시 HTTPS로 받아야 한다. 평문 HTTP면 중간자가 `jwks_uri`를 자기 키로 바꿔치기할 수 있다.
> - **issuer 화이트리스트**: "어떤 OP라도 받아준다"는 멀티 테넌트 앱이라도, 허용 issuer 목록은 명시적으로 두어야 한다. Discovery는 누구나 가짜로 호스팅할 수 있다.
> - **알고리즘 화이트리스트**: `id_token_signing_alg_values_supported`를 그대로 따라가지 말고, **코드에서 기대 알고리즘을 못박아라**. `alg:none` 공격, `RS256 → HS256` 혼동 공격에 대한 기본 방어다.
> - **issuer ↔ iss 정확 일치**: Discovery의 `issuer`와 ID Token의 `iss`가 한 글자도 다르면 안 된다.

### 한 줄 정리

> [!note] 결론
> **Discovery는 `issuer` 하나로 OIDC 통합의 모든 좌표(엔드포인트·키 위치·지원 알고리즘)를 자동 획득하게 해 주는 표준 메타데이터**다. `/.well-known/openid-configuration` → `jwks_uri` → JWKS → 토큰의 `kid` 매칭 → 공개키 검증 → claim 검증으로 이어지는 한 줄짜리 파이프라인이 OIDC 신뢰의 뼈대다.

---

## English

### TL;DR

[[OAuth 2.0]] handles **authorization** — *"may this app access my resources?"* — while [[OIDC]] layered on top handles **authentication** — *"who is this user?"*. The two share almost the same [[Authorization Code Flow]], but the deliverables differ.

> [!note] One-line definition
> **OIDC = OAuth 2.0 + ID Token + standardized user-info lookup.** OAuth issues permission; OIDC issues identity.

| Aspect | [[Access Token]] | [[ID Token]] |
|---|---|---|
| Defined by | [[OAuth 2.0]] | [[OIDC]] |
| Purpose | API / resource access | Proof of user identity |
| Format | Opaque or [[JWT]] (spec leaves it open) | **Always [[JWT]]** |
| Consumer | Resource Server | Client (RP) |
| OK to inspect? | ❌ (client must not parse it) | ✅ (verify, then use sub/email/etc.) |

```
┌─────────────────────────────────────────────────────┐
│   scope=drive.readonly         → Access Token       │  ← OAuth 2.0
│   scope=openid profile email   → Access + ID Token  │  ← OIDC
└─────────────────────────────────────────────────────┘
```

A [[JWT]] is three parts: `Header.Payload.Signature`. Its safety rests on **signing, not encryption** — Header/Payload are plain Base64URL that anyone can decode, and what a JWT guarantees is not confidentiality but **integrity** and **authenticity**. [[OIDC]] defaults to [[RS256]] (asymmetric): the [[OpenID Provider]] **signs with a private key** and clients **verify with the public key** fetched from [[JWKS]]. Without the private key no valid signature can be produced; the public key only verifies — forgery is structurally impossible.

> [!warning] The core weakness of [[Bearer Token]]s
> "Token = authority", so **a stolen token lets the thief impersonate the owner**. Signatures prevent tampering, not wholesale copying. A stolen token is a perfectly valid, real token.

Defense is not one trick but a stack of layers:

```
Block theft         HTTPS, HttpOnly Cookie, SameSite, XSS hardening
Shrink the window   Short Access Token lifetime (5 min ~ 1 hr)
Make it unusable    DPoP, mTLS (bind the token to a key)
Detect & revoke     [[Refresh Token Rotation]] + Reuse Detection
```

[[Refresh Token Rotation]] makes each RT **single-use**: if a revoked RT shows up again, "two parties are using the same chain = theft", and the whole [[Token Family]] is invalidated. Keys themselves rotate too — the [[OpenID Provider]] publishes multiple public keys in [[JWKS]], and the [[kid]] in the token Header picks the right one, so apps survive key rotation with no code change. And every endpoint, key location, and issuer is auto-discovered through the [[OIDC Discovery]] document (`/.well-known/openid-configuration`), so the only thing the app needs to know is the **issuer URL**.

> [!tip] Mnemonic
> [[OIDC Discovery]] = OP's **manual / info desk** · `jwks_uri` = **address of the key locker** · [[JWKS]] = **the key locker** · [[kid]] = **the label on each key**.

### OAuth 2.0 vs OIDC: Separating Authorization from Authentication

These two are often lumped together, but they **solve different problems.** [[OAuth 2.0]] is an **authorization** protocol — "is this app allowed to touch my resources?" [[OIDC]] (OpenID Connect) is an extension on top of it that adds **authentication** — "and *who* exactly is this user?"

### The one-line formula

> [!note] Core equation
> **OIDC = [[OAuth 2.0]] + [[ID Token]] (identity payload, JWT) + a standard way to fetch user info**
>
> OIDC does **not replace** OAuth 2.0. The OAuth flow stays exactly the same — add `scope=openid` and the response gets an extra [[ID Token]] piggybacked on it. OIDC is an **extension layer**, not a rewrite.

### Authorization vs Authentication, side by side

| Aspect | OAuth 2.0 | OIDC |
|--------|-----------|------|
| Purpose | **Authorization** | **Authentication** |
| Question it answers | "Can this app access my resources?" | "Who is this user?" |
| Output | [[Access Token]] (+ Refresh Token) | [[ID Token]] + Access Token (+ Refresh Token) |
| Token format | Access Token format unspecified (opaque or JWT both OK) | ID Token is **always a JWT** (spec-mandated) |
| Canonical use case | "Let my app read your Google Drive" | "Sign in with Google" |
| Released | 2012 (RFC 6749) | 2014 (OpenID Foundation) |

> [!tip] An intuitive analogy
> - **OAuth 2.0** = "Hotel keycard issuance" — holding the keycard means you can enter that room. **The card doesn't tell anyone whose card it is.**
> - **OIDC** = "ID check at check-in + keycard issuance" — your ID document ([[ID Token]]) proves *who* you are; the keycard ([[Access Token]]) controls *what* you can access.

### Why OIDC was needed: an Access Token alone allows identity forgery

OAuth 2.0's [[Access Token]] is designed to carry exactly one meaning: *"the bearer of this token has these permissions."* That gave rise to a classic **confused-deputy** problem when people tried to use OAuth for login.

> [!warning] Identity forgery when OAuth 2.0 is used for sign-in
> Many early "Sign in with Facebook"-style implementations treated the OAuth 2.0 Access Token as proof of identity. The flow looked like this:
>
> 1. App receives an `access_token` for the user
> 2. App calls a nonstandard `GET /me?access_token=...` endpoint to get the user's ID
> 3. App logs them in as that ID
>
> **The attack:** an attacker grabs an Access Token issued for their own (malicious) app **X**, then submits it to legitimate app **Y**. App Y calls `/me`, gets back "this is user A," and logs the attacker in as user A. **But that token was issued for app X, not app Y.** The Access Token contains no verifiable claim about *which app it was issued to*. This is the [[Token Substitution Attack]] — the confused deputy problem.

[[OIDC]] addresses this head-on. The [[ID Token]] is a JWT containing two claims that are **cryptographically verifiable**:

- `iss` (issuer): who minted this token (e.g. `https://accounts.google.com`)
- `aud` (audience): **who** this token was minted **for** (e.g. your app's `client_id`)

Your app verifies that `aud` matches its own `client_id` and verifies the signature against the OP's public key via [[JWKS]]. Only when both pass can you trust that "this token was issued by the real OP, specifically for our app."

```
With OAuth 2.0 alone (vulnerable):
  Access Token for app X ──> app Y's /login endpoint
                                    │
                                    └─> No way to know "who this token belongs to"
                                        Bearer == user A → forgery succeeds

With OIDC (safe):
  ID Token for app Y ──> app Y verifies:
                          - Signature OK?  (JWKS public key)
                          - iss == "https://accounts.google.com"?
                          - aud == "app Y's client_id"?    ← rejects tokens for other apps
                          - exp not expired?
                          - nonce matches?
                         All pass → "this really is user A" confirmed
```

> [!example] The flow is nearly identical — only the request differs
>
> **OAuth 2.0 (delegate Google Drive read):**
> ```
> GET /authorize?response_type=code
>                &client_id=...
>                &scope=https://www.googleapis.com/auth/drive.readonly
>                &redirect_uri=...
>                &state=...
> → Result: Access Token (for Drive API)
> ```
>
> **OIDC (sign in with Google):**
> ```
> GET /authorize?response_type=code
>                &client_id=...
>                &scope=openid profile email     ← "openid" is the OIDC switch
>                &redirect_uri=...
>                &state=...
>                &nonce=...                       ← added by OIDC
> → Result: ID Token + Access Token
> ```
>
> Difference: two lines. **Is `openid` in `scope`?** And does the response carry an [[ID Token]]? That's why OIDC is described as "a thin authentication layer riding on OAuth."

### Which one should you use?

> [!tip] Decision rule
> - **You need login** (you must identify the user) → [[OIDC]]
> - **You need to call another service's API on the user's behalf** (read Drive, write Calendar, etc.) → [[OAuth 2.0]]
> - **You need both** ("Sign in with Google" *and* read Drive) → a **single OIDC flow** gets you the ID Token and Access Token together

For social login, [[SSO]], and B2B authentication — anywhere identifying the user is the point — OIDC is the default. When the point is purely API delegation, plain OAuth 2.0 suffices, though in practice you almost always want to know who the user is too, so most "social login" flows you'll encounter in the wild are really OIDC.

### OAuth 2.0 Authorization Code Flow

OAuth 2.0's Authorization Code Flow answers the question: "How can my app access a user's [[google-drive|Google Drive]] on their behalf?" The core idea is that **the app never sees the user's password, yet still obtains a scoped token to act on the user's behalf**.

### The Four Actors

You have to nail down the roles. OAuth 2.0 defines exactly four:

| Role | Who | What they do | Example |
|------|-----|--------------|---------|
| **Resource Owner** | The user themselves | Actual owner of the data; the one delegating access | Me (a Google account holder) |
| **Client** | The client application | Third-party app trying to access the user's resources | A backup app I built |
| **Authorization Server** | Auth/identity server | Logs the user in, collects consent, issues tokens | accounts.google.com |
| **Resource Server** | The API server | Validates Access Tokens and returns actual data | www.googleapis.com/drive |

> [!note] Authorization Server and Resource Server are operated by the same company, but they are distinct roles by spec
> Google runs both, but the responsibilities of issuing tokens (AS) and serving data (RS) are separated. That's why one AS can issue a token usable across multiple RSes (Drive, Gmail, Calendar).

### The Flow at a Glance

```
┌─────────┐         ┌────────┐         ┌──────────────┐         ┌──────────────┐
│  User   │         │ Client │         │ Auth Server  │         │ Resource     │
│ (Owner) │         │  App   │         │  (Google)    │         │ Server (API) │
└────┬────┘         └───┬────┘         └──────┬───────┘         └──────┬───────┘
     │  ① "Link Drive"   │                    │                        │
     │ ─────────────────> │                    │                        │
     │                    │  ② Redirect to /authorize                   │
     │                    │  client_id, redirect_uri,                   │
     │                    │  scope=drive.readonly, state, PKCE          │
     │ <─────────────────────────────────────  │                        │
     │   ③ Login + consent screen              │                        │
     │ <────────────────────────────────────>  │                        │
     │                    │                    │                        │
     │                    │  ④ authorization code (single-use, ~10 min) │
     │                    │ <───── redirect_uri?code=AUTH_CODE ──────── │
     │                    │                    │                        │
     │                    │  ⑤ POST /token                              │
     │                    │  code + client_id + client_secret           │
     │                    │ ─────────────────> │                        │
     │                    │                    │                        │
     │                    │  ⑥ Access Token (+ Refresh Token)           │
     │                    │ <───────────────── │                        │
     │                    │                    │                        │
     │                    │  ⑦ GET /drive/files                         │
     │                    │  Authorization: Bearer <AccessToken>        │
     │                    │ ──────────────────────────────────────────> │
     │                    │                    │                        │
     │                    │  ⑧ File listing JSON                        │
     │                    │ <────────────────────────────────────────── │
```

### Step by Step

#### ① User clicks the link button
The user clicks a "Connect Google Drive" button and the client app starts the flow.

#### ② Redirect to /authorize (Front Channel)
The app redirects the user's browser to the [[authorization-endpoint|Authorization Server's /authorize]] endpoint. The key thing: **the app does not call this directly — it sends the user's browser there**.

```
https://accounts.google.com/o/oauth2/v2/auth?
  response_type=code
  &client_id=12345.apps.googleusercontent.com
  &redirect_uri=https://myapp.com/callback
  &scope=https://www.googleapis.com/auth/drive.readonly
  &state=xyzABC123             ← random value for CSRF protection
  &code_challenge=...          ← PKCE
  &code_challenge_method=S256
```

> [!example] What scope=drive.readonly means
> `scope` is the set of permissions the app is asking for. `drive.readonly` means "let me only **read** the user's Drive files." It differs from `drive` (full read/write) or `drive.file` (only files the app itself created). The user consents to exactly this scope on the consent screen.

#### ③ User login + consent screen
The browser shows Google's login page. After login, a consent screen says: "This app wants to **read** your Drive files. Allow?"

> [!tip] The password is only ever entered on Google's domain
> This step is OAuth's core value. The user enters their password at `accounts.google.com` — **never** to the [[client|Client App]]. Even if the app is malicious, the password stays safe.

#### ④ Authorization Code issued
Once consent is given, the Auth Server redirects the browser back to `redirect_uri` with a `code` attached.

```
https://myapp.com/callback?code=4/0AeaY...AUTH_CODE&state=xyzABC123
```

This `code` is:
- **Single-use** (discarded after one exchange)
- **Short-lived** (typically 10 minutes)
- **Transits the front channel but isn't a token itself** → useless to a thief without `client_secret`

The app verifies the returned `state` matches the one it sent in ② to defend against CSRF.

#### ⑤ /token exchange (Back Channel)
Now the **app's backend** directly calls the Auth Server's `/token`. This call does not go through the browser.

```http
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=4/0AeaY...AUTH_CODE
&client_id=12345.apps.googleusercontent.com
&client_secret=GOCSPX-xxxxx            ← secret only backend knows
&redirect_uri=https://myapp.com/callback
&code_verifier=...                      ← PKCE
```

> [!warning] Never expose client_secret to the frontend
> If `client_secret` leaks, anyone can impersonate your app in the token exchange. SPAs and mobile apps are "public clients" and can't safely hold a secret — they use [[pkce|PKCE]] instead.

#### ⑥ Receive Access Token (+ Refresh Token)
The Auth Server returns the tokens as JSON.

```json
{
  "access_token": "ya29.a0AfH6...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "1//0gK7...",
  "scope": "https://www.googleapis.com/auth/drive.readonly"
}
```

- **Access Token**: for API calls (short-lived, ~1 hour)
- **Refresh Token**: to renew Access Tokens (long-lived, days to weeks)
- **scope**: the permissions actually granted (may differ from what was requested)

#### ⑦ Call the API with the Access Token
Now the app calls the Resource Server using the token.

```http
GET /drive/v3/files HTTP/1.1
Host: www.googleapis.com
Authorization: Bearer ya29.a0AfH6...
```

`Bearer` means "whoever bears this token gets the permission." The Resource Server validates the token (signature check for JWT, introspection call to the AS for opaque tokens) and returns scope-appropriate data.

#### ⑧ Resource returned
The Drive API returns the file listing as JSON. Done.

### Front Channel vs Back Channel

The security of this flow rests on the fact that **sensitive material never traverses the browser**.

| Channel | What flows | Risk |
|---------|-----------|------|
| **Front Channel** (browser redirects) | client_id, scope, state, **authorization code** | Code is single-use and useless without client_secret |
| **Back Channel** (app backend ↔ Auth Server, direct) | client_secret, **Access Token**, **Refresh Token** | TLS-encrypted, never exposed externally |

> [!note] This is exactly why Implicit Flow was retired
> The old Implicit Flow returned an Access Token directly via the redirect URI fragment (`#access_token=...`). Tokens leaked through browser history and Referer headers too often. OAuth 2.1 removes it entirely. Every client should use **Authorization Code + [[pkce|PKCE]]**.

### Key Insight

> [!tip] OAuth 2.0 gives you "permission," not "identity"
> When this flow ends, the app knows only that "this Access Token can exercise drive.readonly." It **does not know who received the token (who the user is).** That limitation is exactly why OIDC exists — OIDC adds `scope=openid` to the same flow and returns an additional [[id-token|ID Token]] to fill the gap (see [[oidc-flow|the OIDC flow]]).

### OIDC Flow and Token Types

The [[OIDC]] flow follows the [[OAuth 2.0]] [[Authorization Code Flow]] almost verbatim. The single decisive difference is whether the `scope` parameter contains the magic word **`openid`**. The moment that word is present, the [[Authorization Server]] expands its role from "place that hands out permissions" into "place that vouches for identity" (the [[OpenID Provider|OP]]), and slips an extra [[ID Token]] into the response.

### 1. Same flow, with `openid` layered on top

The flow is identical; one line in the request and one line in the response differ.

```
[OAuth 2.0 — authorization only]
GET /authorize?
    response_type=code
    &client_id=app123
    &redirect_uri=https://app.example/cb
    &scope=drive.readonly                ← pure OAuth scope
    &state=xyz

→ exchange at /token →

{ "access_token": "ya29.a0Af...", "refresh_token": "1//0g..." }
   └── No ID Token. You cannot tell "who the user is" from this.


[OIDC — authorization + authentication]
GET /authorize?
    response_type=code
    &client_id=app123
    &redirect_uri=https://app.example/cb
    &scope=openid profile email          ← the word "openid" switches OIDC on
    &state=xyz
    &nonce=n-0S6_WzA2Mj                  ← replay protection (required by OIDC)

→ exchange at /token →

{
  "access_token":  "ya29.a0Af...",       ← defined by OAuth
  "id_token":      "eyJhbGciOi...",      ← extra one added by OIDC (a JWT)
  "refresh_token": "1//0g...",
  "token_type":    "Bearer",
  "expires_in":    3600
}
```

> [!note] One-liner
> **OIDC = the OAuth 2.0 flow, with `scope=openid` added, returning one extra [[ID Token]].** It is not a new protocol — it is an authentication layer riding on top of OAuth.

### 2. Three tokens, three separate responsibilities

The `/token` response carries three things, and **each has a different purpose, a different format, and a different audience.** Pin them down in one table so they stop blurring together.

| Token | Defined by | Purpose | Format | Who reads it | Lifetime |
|---|---|---|---|---|---|
| **[[Access Token]]** | [[OAuth 2.0]] | API (resource) access | **Spec leaves it open** — can be opaque, can be a [[JWT]] | [[Resource Server]] | Short (5 min – 1 h) |
| **[[ID Token]]** | [[OIDC]] | Proof of "who the user is" | **Always a [[JWT]]** (the spec mandates it) | [[Relying Party\|RP]] (the client app) | Short (typically similar to access token) |
| **[[Refresh Token]]** | [[OAuth 2.0]] | Renew the access token | Usually opaque | [[Authorization Server]] | Long (days to weeks) |

> [!warning] Two things people routinely get wrong
> 1. **"The ID Token uses JWT" is imprecise.** The ID Token *is* a JWT — there is no format choice. The OIDC spec nails it down.
> 2. **Access tokens may or may not be JWTs.** A [[Google]] access token (`ya29.a0Af...`) is an opaque blob; an [[Auth0]]/[[Keycloak]] access token is usually a JWT. **The client must not crack open an access token** — that token exists to be flung at a Resource Server, not parsed by the client.

The audiences become obvious once drawn:

```
                      ┌──────────────────────────┐
                      │   Authorization Server   │
                      │   (the OP, e.g. Google)  │
                      └────────────┬─────────────┘
                                   │ /token response
                  ┌────────────────┼────────────────┐
                  ▼                ▼                ▼
           ┌────────────┐   ┌────────────┐   ┌──────────────┐
           │ ID Token   │   │AccessToken │   │RefreshToken  │
           │ (JWT)      │   │(opaque/JWT)│   │(opaque)      │
           └─────┬──────┘   └─────┬──────┘   └──────┬───────┘
                 │                │                 │
                 ▼                ▼                 ▼
        The app (Relying    The Resource         Sent back only
        Party) verifies it  Server (Google       to the Authorization
        itself to know      Drive API)           Server to mint a new
        "who the user is"   verifies it          access token
```

> [!tip] Different responsibility ⇒ different verifier
> An [[ID Token]] is **verified by the app itself** (signature + iss/aud/exp/nonce). An [[Access Token]] is **verified by the Resource Server, not the app**. That is why the client has no reason to peek inside an access token even when it happens to be a JWT — that letter wasn't addressed to it.

### 3. The mandatory ID Token claims — `iss / sub / aud / exp / iat / nonce`

The [[ID Token]] is a [[JWT]], so it is `Header.Payload.Signature`, and OIDC nails down **six claims that must appear in the payload**.

```json
// ID Token payload — what you get after base64url-decoding it
{
  "iss":   "https://accounts.google.com",          // Issuer
  "sub":   "10769150350006150715113082",            // Subject (stable user ID)
  "aud":   "app123.apps.googleusercontent.com",     // Audience = my client_id
  "exp":   1750800000,                               // Expiration
  "iat":   1750796400,                               // Issued At
  "nonce": "n-0S6_WzA2Mj",                           // the nonce I sent at /authorize

  // Below are optional, granted via scope
  "email":          "justin.seo@mvlchain.io",
  "email_verified": true,
  "name":           "Justin Seo",
  "picture":        "https://lh3.googleusercontent.com/..."
}
```

Best learned by **what breaks if each one is missing**:

| Claim | Meaning | What you check | What breaks if omitted |
|---|---|---|---|
| `iss` | Issuer — who minted the token | Must match the exact issuer string of the OP you trust | You'd accept tokens minted by some other OP |
| `sub` | Subject — stable user ID inside that OP | Use it as the user key in your DB | You cannot identify the same human across logins |
| `aud` | Audience — who this token is for (= your `client_id`) | Must equal your `client_id` | A [[Confused Deputy]]: a token issued for the app next door gets accepted by yours |
| `exp` | Expiration (Unix timestamp) | "Now" must be strictly less than `exp` | Expired tokens live forever |
| `iat` | Issued At | Reject future-dated and ancient tokens | No defense against clock skew / replay corroboration |
| `nonce` | The [[nonce]] you sent at `/authorize`, echoed back | Must match the nonce you actually sent | No defense against a [[Replay Attack]] reusing an ID Token |

> [!example] Identify the user by `iss + sub`, never `sub` alone
> `sub` is unique **only inside one OP**. Google's `sub=123` and Apple's `sub=123` are different humans. Always key your user table by the **`(iss, sub)` pair**. If one human uses both Google and Apple login, you'll have two rows — collapsing them into one person is a separate account-linking concern.

> [!warning] Do not skip `aud` validation
> When several apps register against the same OP and receive tokens signed by the same key, **forgetting to validate `aud` lets your app accept the neighbor app's token**. The signature is real, the issuer matches — every other check passes. The one-line `aud === my_client_id` check is the only thing standing in the way.

> [!tip] Suggested verification order
> ① Signature ([[JWKS]] lookup by `kid`) → ② `iss` → ③ `aud` → ④ `exp`/`iat` → ⑤ `nonce`. If the signature fails, no claim is trustworthy, so end the check there. Most libraries ([[jose]], [[jsonwebtoken]], [[python-jose]]) do this order automatically — but **`audience` and `issuer` are only validated if you pass them in explicitly** at the call site.

### JWT Signature Structure & Verification: What Signs What

The safety of a [[JWT]] rests **not on encryption but on the signature**. Misunderstand this single point and the whole security story collapses.

> [!warning] JWTs are NOT encrypted
> The Header and Payload are merely **Base64URL-encoded**; anyone can decode them to plaintext. What a JWT guarantees is **integrity and authenticity, not confidentiality**. Never put passwords, SSNs, or payment data inside the Payload.

### 1. The 3-part Structure: Header.Payload.Signature

A JWT is three chunks joined by dots (`.`):

```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEyMyJ9 . eyJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2dsZS5jb20iLCJzdWIiOiIxMTAxNjkiLCJhdWQiOiJteS1jbGllbnQtaWQiLCJleHAiOjE3MTk4NDgwMDB9 . NHVB2tQs...XyZ
└────────── Header ──────────┘ └─────────────────── Payload ───────────────────┘ └ Signature ┘
```

| Part | Contents | Example |
|---|---|---|
| **Header** | algorithm (`alg`), key id (`kid`), type (`typ`) | `{"alg":"RS256","kid":"abc123","typ":"JWT"}` |
| **Payload** | Claims (`iss`, `sub`, `aud`, `exp`, `iat`, `nonce`, ...) | `{"iss":"https://accounts.google.com","sub":"110169","aud":"my-client-id","exp":1719848000}` |
| **Signature** | result of signing Header+Payload with a key | binary, then base64url-encoded |

### 2. What gets signed: the `signing_input`

The signature does NOT cover Header or Payload alone — it covers the **dot-joined concatenation of both**.

```
signing_input = base64url(Header) + "." + base64url(Payload)
Signature     = Sign( signing_input, key )
```

Flip a single character in `alg`, or in `sub`, and the signature no longer matches. **Tampering = instantly detected**.

> [!example] Sign / Verify pseudocode
> ```python
> # Signing (issuer)
> header_b64  = base64url(json.dumps({"alg":"RS256","kid":"abc123"}))
> payload_b64 = base64url(json.dumps({"iss":..., "sub":..., "exp":...}))
> signing_input = f"{header_b64}.{payload_b64}".encode()
> signature_b64 = base64url(RSA_Sign(signing_input, private_key))
> jwt = f"{header_b64}.{payload_b64}.{signature_b64}"
>
> # Verifying (relying party)
> h, p, s = jwt.split(".")
> signing_input = f"{h}.{p}".encode()
> assert RSA_Verify(signing_input, base64url_decode(s), public_key)  # integrity
> # Then validate claims (iss / aud / exp / nonce ...)
> ```

### 3. Two signing families: symmetric vs asymmetric

| Aspect | Symmetric ([[HMAC]]) | Asymmetric ([[RSA]] / [[ECDSA]]) |
|---|---|---|
| **Algorithms** | `HS256`, `HS384`, `HS512` | `RS256`, `RS384`, `RS512`, `ES256`, `ES384`, `EdDSA` |
| **Key material** | one shared `secret` | `private_key` + `public_key` pair |
| **Signing** | HMAC with the secret | sign with the private key |
| **Verification** | recompute HMAC with the SAME secret | verify with the public key |
| **Speed** | very fast | slower (RSA in particular) |
| **Public verification** | impossible — anyone with the secret can also forge | possible — the public key cannot forge |
| **Best fit** | self-contained session tokens (issuer == verifier) | [[OIDC]], distributed verification, external clients |

> [!note] OIDC assumes asymmetric signing
> OPs like Google, Apple, [[Auth0]], [[Keycloak]], [[Okta]] sign with **`RS256` (or `ES256`)** and publish only the public keys via [[JWKS]] to the entire world. The **private key never leaves the OP**, so forgery is structurally impossible; any verifier checks integrity with the public key alone. This asymmetry is exactly why trust can hold when issuer and verifier are separated.

#### HS256 (symmetric) flow

```
        ┌──────────┐    secret    ┌──────────┐
App A ──►│ Sign HMAC├──────────────┤Verify HMAC│◄─── App A (itself)
        └──────────┘   (shared)   └──────────┘
                  ⚠ anyone who knows the secret can forge
```

#### RS256/ES256 (asymmetric) flow

```
   OP (Authorization Server)              Relying Party (Client/API)
  ┌────────────────────────┐             ┌──────────────────────────┐
  │  Private Key  ──Sign──►│  ID Token   │  Public Key (from JWKS)  │
  │  (OP only)             │ ──────────► │  ──Verify──► OK / NG     │
  └────────────────────────┘             └──────────────────────────┘
        ✅ Cannot produce a valid signature without the private key
        ✅ Public keys can only verify, never forge
```

### 4. RS256 verification, step by step

1. **Decode the JWT** → extract `header`, `payload`, `signature`.
2. **Whitelist the `alg`** → only the algorithm you expect (e.g. `RS256`, NEVER `none`).
3. **Extract `kid`** → decides which public key to use.
4. **Fetch the public key from JWKS** → from [[jwks_uri]], cached; refetch on cache miss (details in the [[JWKS]] / [[kid]] section).
5. **Verify the signature** → `RSA-Verify(signing_input, signature, public_key)`.
6. **Validate claims** → `iss` (issuer), `aud` (= your `client_id`), `exp` (expiry), `nbf`, `iat` (within clock skew), `nonce` (matches what you stored).

> [!tip] Stick to that order, top to bottom
> If the signature fails, **nothing in the Payload is trustworthy**. Code that picks the key based on `iss` or `aud` from the payload is dangerous. Always **pick the key by Header's `kid` only, and read claims only after the signature has passed.**

### 5. Attacks and defenses

#### ① Naive tampering

Attacker flips `"role":"user"` to `"role":"admin"` in the payload.

```
original: base64url(H) . base64url({"role":"user"})  . sig
tampered: base64url(H) . base64url({"role":"admin"}) . sig   ← signing_input changed → sig no longer matches
```

→ Signature verification fails immediately. **Done.**

#### ② The `alg: none` attack

The [[JWT]] spec famously included a `"alg":"none"` (unsigned) mode. If a library accepts it blindly, an attacker can send:

```json
Header  : {"alg":"none"}
Payload : {"sub":"admin"}
Signature: (empty)
```

If the server reads `alg` straight from the token — "ah, no signature needed, accept!" — game over.

> [!warning] Don't read `alg` from the header — pin it on the verifier
> Verification code must **whitelist allowed algorithms**:
> ```python
> # ❌ Dangerous: trust the header's alg
> jwt.decode(token, key, algorithms=[header["alg"]])
>
> # ✅ Safe: only what we expect
> jwt.decode(token, key, algorithms=["RS256"])  # "none" rejected automatically
> ```

#### ③ Algorithm confusion (RS256 → HS256)

A server expects RS256-signed tokens but its verification call does not pin the algorithm. An attacker:

1. Changes Header's `alg` from `RS256` to **`HS256`**.
2. Uses the **public RSA key bytes themselves as the HMAC secret** to compute a new signature.
3. Server reads `alg` from the header, blindly verifies as HS256, using the same public key as the HMAC secret → HMAC matches → accepted.

```
Attacker's forged token:
  Header  : {"alg":"HS256"}   ← changed from RS256
  Payload : {"sub":"victim"}
  Sig     : HMAC-SHA256(signing_input, RSA_public_key_bytes)
                                       └──── publicly downloadable ────┘
Server (mistake):
  Reads alg from header → HS256
  Reuses the "same key material", which is actually the RSA public key
  → HMAC matches → accepted 😱
```

Defense is the same: **pin `algorithms=["RS256"]`**, and use libraries that force the key object to a matching algorithm type ([[node-jsonwebtoken]], [[PyJWT]], [[jose]] all offer such options).

#### ④ Key confusion / key leak

- If the private key leaks, **every signature can be forged**. → Immediate [[Key Rotation]] (see the [[JWKS]] section).
- Separate keys per environment (dev/staging/prod), and keep them in an [[HSM]] or [[KMS]].

### 6. Algorithm quick reference

| `alg` | Family | Key/curve | Notes |
|---|---|---|---|
| `HS256` | HMAC-SHA256 (symmetric) | secret ≥ 256-bit | Fast; single-trust-domain only |
| `RS256` | RSA-PKCS1-v1_5 + SHA-256 | RSA 2048–4096 | De-facto OIDC default; best compat |
| `RS384`, `RS512` | variants of RS256 | same | Different hash length |
| `ES256` | ECDSA + P-256 + SHA-256 | EC P-256 | Short signatures (~64B), fast verify, mobile-friendly |
| `ES384`, `ES512` | ECDSA P-384/P-521 | EC | Stronger security |
| `EdDSA` (Ed25519) | Edwards-curve | 256-bit | Most modern; deterministic signing |
| `PS256` | RSASSA-PSS + SHA-256 | RSA | Safer padding (random salt) |
| `none` | no signature | — | **Forbidden** |

> [!tip] Choosing today? Prefer ES256 or EdDSA
> RS256 sticks around for compatibility. For greenfield designs, **ES256 (P-256)** or **EdDSA (Ed25519)** generally win — shorter signatures, faster verification.

### 7. The one-liner

> [!note] Core takeaway
> OIDC **signs with a private key** and **verifies with the JWKS public key**. Because the signature covers both Header and Payload, any tampering breaks immediately, and forgery is impossible without the private key. The last line of defense is one rule: **"Do not read `alg` from the header — pin it on the verifier."**

### Bearer Token Theft and Defense Strategy

### 1. What Is a Bearer Token — The "Possession = Authority" Trap

The tokens used by OAuth 2.0 and [[OIDC]] are mostly **Bearer Tokens**. As the name says, "whoever *bears* the token is the authorized party." The server only checks whether the attached token is valid — it does not ask "is the sender of this request actually the original owner?"

```
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEyMyJ9.eyJzdWI...
```

> [!warning] Core Analogy: A Bearer Token Is **Cash**
> A $10 bill is worth $10 no matter who holds it. The cashier does not ask "who originally owned this bill?" A Bearer Token works the same way. Whoever shows up with it, the server says "valid token, OK" and lets them through.

So **the moment it's stolen, the attacker can impersonate the original owner**. This is the most fundamental weakness of token-based authentication, and the answer to "if the token is stolen, can the attacker impersonate me?" is **unambiguously yes**.

---

### 2. Why Signatures Can't Stop Theft

JWT's [[JWT Signature]] is strong. So why doesn't it stop theft?

| What the signature **does** stop | What the signature **does not** stop |
|---|---|
| Changing one byte of payload → signature breaks → rejected | **Copying the whole token verbatim** and replaying it |
| Forging a brand-new token → no private key → impossible | "Who is the current sender?" — the signature only validates *content*, not the *holder* |
| `alg:none` style algorithm tricks → blocked via whitelist | Real tokens leaked via XSS, MITM, log spillage, etc. |

> [!note] A stolen Bearer Token Is a **Perfectly Valid Real Token**
> Signature: fine. `exp`: not yet. `iss`/`aud`: match. To the server, the "legit request" and the "theft request" are **cryptographically indistinguishable**. That's the whole problem.

---

### 3. Defense-in-Depth Philosophy — "You Can't Bring Theft to 0%"

No single technique blocks token theft completely. So OIDC security takes a **Defense-in-Depth** approach. In one line:

> [!tip] OIDC Security Philosophy
> **"Make theft hard + minimize damage when stolen + detect and revoke fast."**
> No single layer is allowed to be the lone wall. Stack walls so no attack succeeds by piercing just one.

```
┌────────────────────────────────────────────────────────────┐
│  ① Prevent Theft    : HTTPS, HttpOnly Cookie, CSP, SameSite │
│  ② Limit Damage     : Short Access Token TTL (5–60 min)     │
│  ③ Break Possession : DPoP, mTLS (Sender-Constrained)       │
│  ④ Detect & Revoke  : Refresh Rotation, Blacklist, Anomaly  │
└────────────────────────────────────────────────────────────┘
```

Let's walk each axis.

---

### 4. ① Short-Lived Tokens

The simplest, most effective defense. **Even if stolen, the token expires soon.**

| Token type | Recommended TTL | Why |
|---|---|---|
| **Access Token** | 5 min – 1 hour | Exposed on every request. High leak surface. Keep short. |
| **ID Token** | 5 min – 1 hour | Usually validated once at login and discarded. |
| **Refresh Token** | Days to weeks | Can be stored safely. Protected by Rotation (see [[Refresh Token Rotation]]) |

> [!example] Why short lifetimes help
> Suppose Access Token TTL = 15 min. An attacker who steals it via [[XSS]] gets **at most 15 minutes of abuse**. Logout or token rotation during that window cuts it shorter. Compared to a 24-hour token, the attack window shrinks by 96×.

---

### 5. ② Transport & Storage Protection

Close the channels through which a token could leak.

#### Force HTTPS
- Sending tokens over plaintext HTTP exposes them to Wi-Fi sniffing and [[MITM]] attacks.
- The OIDC spec mandates HTTPS on every endpoint (`authorization_endpoint`, `token_endpoint`, `jwks_uri`, etc.).

#### HttpOnly + Secure + SameSite Cookies
Where to store tokens in an SPA is a major issue. Options and risks:

| Storage | Stolen by XSS? | CSRF risk | Recommended |
|---|---|---|---|
| `localStorage` | **Trivial to steal** (JS can read) | None | ✗ Avoid |
| `sessionStorage` | **Trivial to steal** | None | ✗ Avoid |
| In-memory JS variable | Stolen if XSS injects into the page | None | △ Access Token only |
| **HttpOnly Cookie** | **JS cannot read it** | Mitigated by SameSite | ✓ Refresh Token |

> [!tip] The Decisive Win of HttpOnly Cookies
> Not even `document.cookie` can read them. The browser automatically attaches the cookie to requests, but JavaScript never sees the value. Even with [[XSS]], the token does not leak in bulk. Pair it with `Secure` (HTTPS only) + `SameSite=Lax/Strict` ([[CSRF]] protection) to complete the picture.

```http
Set-Cookie: refresh_token=eyJhbGc...; HttpOnly; Secure; SameSite=Strict; Path=/auth
```

---

### 6. ③ Token Binding — Break the "Possession = Authority" Equation

The most fundamental fix. Stamp the token with **"only a client that possesses key X may use this token,"** so a thief without the key gets nothing. Such tokens are called **Sender-Constrained Tokens**.

#### DPoP (Demonstrating Proof-of-Possession) — RFC 9449

The client generates a key pair and **attaches a fresh short-lived proof JWT (DPoP Proof), signed with its private key, on every API call.**

```http
POST /api/resource
Authorization: DPoP eyJhbGc...           ← Access Token
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIs...        ← Proof freshly signed by the client
```

- The server verifies two things:
  1. The Access Token is valid.
  2. The DPoP Proof is signed by the public key whose thumbprint (`jkt`) is bound into the token.
- If the attacker steals the Access Token but not the private key → they cannot forge the DPoP Proof → **rejected**.

#### mTLS (Mutual TLS) — RFC 8705

At the TLS handshake, **the client also presents a certificate**. The Access Token embeds that certificate's thumbprint (`x5t#S256`). Using the same token with a different certificate → rejected.

> [!warning] DPoP / mTLS Are Powerful But Costly
> Implementation complexity, key management, and client compatibility burden are real. They're typically deployed in **finance / high-value APIs** (e.g. PSD2 Open Banking). Most consumer web apps get by with ①②④. But if you want "stolen but unusable," this is the surest answer.

---

### 7. ④ Detection & Revocation

The axis for catching theft fast and cutting it off.

- **[[Refresh Token Rotation]] + Reuse Detection**: make refresh tokens single-use; the moment a revoked token reappears, treat it as theft and invalidate the whole token family. (Covered in its own section.)
- **Anomaly Detection**: same token used simultaneously in Seoul and New York, sudden User-Agent change, IP range jump → force re-authentication.
- **Blacklist (Revocation List)**: on logout, password change, or suspected theft, put the token's `jti` into [[Redis]] and check it on every request.
- **Step-Up Authentication**: sensitive actions (money transfer, account deletion) demand additional [[MFA]] even with a valid token.

#### The Revocation Dilemma for JWTs

> [!warning] JWT's Double-Edged Sword: Self-Containment
> JWTs verify via signature alone, so **the server doesn't have to hit the DB on every request** (performance win). But the flip side: the server has no native way to declare "this token is now invalid." You either wait for `exp` or maintain a blacklist — and the blacklist forces a [[Redis]] lookup per request, partly undoing JWT's perf benefit.
>
> Practical compromise: **Keep Access Tokens short (no blacklist; just wait for expiry) + only blacklist Refresh Tokens (lower frequency).** Short lifetimes effectively *are* the revocation strategy.

---

### 8. Putting It Together — One-Page Recap

```
Theft scenario               ↓ Defense layer
────────────────────────────────────────────────────────
Sniffing (Wi-Fi / MITM)  →  ① HTTPS
XSS-based theft          →  ② HttpOnly Cookie + CSP
                            ②' Split Refresh (Cookie) vs Access (memory)
Long-term abuse          →  ③ Short Access Token TTL
Stolen but unusable      →  ④ DPoP / mTLS (token binding)
Refresh Token theft      →  ⑤ Rotation + Reuse Detection
                            ⑥ Wipe entire token family
Detection                →  ⑦ Anomaly detection + Blacklist
Sensitive actions        →  ⑧ Step-Up MFA
```

> [!tip] One-Line Takeaway
> The Bearer Token's strength — "possession = authority" — is also its weakness. Since no single technique is perfect, OIDC security stacks four axes at once: **short lifetimes + safe storage + binding + fast revocation**, simultaneously shrinking *theft probability × abuse duration × blast radius*. That is the core philosophy.

### Refresh Token Rotation and Reuse Detection

[[Refresh Token]] Rotation turns one simple rule — "every Refresh Token is single-use; using it immediately replaces it with a new one" — into an active defence against the worst weakness of [[Bearer Token]]-based auth: long-lived token theft. Its power comes from combining three things: *single-use tokens, server-side history tracking, and family-wide invalidation.*

### Why Refresh Tokens are the most dangerous asset

[[Access Token]]s are short-lived (5 min – 1 hour). [[Refresh Token]]s live for days or weeks. If an attacker steals a Refresh Token, they can *mint fresh Access Tokens at will* until the RT expires. That is a far bigger incident than losing one Access Token.

> [!warning] A Refresh Token is an "Access Token printer"
> Stolen Access Token = 30-minute incident. Stolen Refresh Token = days or weeks of continuous access. Refresh Tokens therefore need much stronger handling than Access Tokens.

### Without rotation: the fixed-RT trap

```
[Login] → RT_AAA issued (valid 30 days)
  /token (RT_AAA) → new AT + same RT_AAA   ← reused forever
  /token (RT_AAA) → new AT + same RT_AAA
  /token (RT_AAA) → new AT + same RT_AAA
  ...
```

If the attacker steals `RT_AAA` at any point:

- They can mint Access Tokens until the RT expires
- The legitimate user keeps using the same `RT_AAA` too, so the server has **no signal to tell the two apart**
- Detection is essentially impossible — the breach is only discovered when the user notices something wrong

### With rotation: single-use Refresh Tokens

Every refresh issues a brand-new RT and immediately retires the previous one.

```
[Login] → RT_AAA  (state=ACTIVE)
  /token (RT_AAA) → new AT + RT_BBB
                    ↳ RT_AAA: state=ROTATED (retired, unusable)
  /token (RT_BBB) → new AT + RT_CCC
                    ↳ RT_BBB: state=ROTATED
  /token (RT_CCC) → new AT + RT_DDD
                    ↳ RT_CCC: state=ROTATED
  ...
```

Each Refresh Token is now valid *exactly once*. Even if stolen, it becomes useless the moment it's used.

### Reuse Detection: when a retired RT reappears

The real power of rotation shows up the second time a retired token is presented.

> [!example] Theft scenario — two parties holding the same chain
> 1. The legitimate user is holding `RT_BBB`
> 2. An attacker steals `RT_BBB` (XSS, MITM, leaked backup, …)
> 3. The attacker calls `/token (RT_BBB)` first → receives new AT + `RT_CCC`. The server marks `RT_BBB` as ROTATED
> 4. Later, the user's Access Token expires and their client tries to refresh with the `RT_BBB` it still holds
> 5. Server: "Hold on — `RT_BBB` is already ROTATED and it's being used again." ← **Reuse Detection fires**

A retired Refresh Token showing up again is unambiguous proof that *two parties are holding the same chain*. It cannot happen in a normal flow.

### Token Family: invalidate at the family level

All Refresh Tokens derived from one login session belong to a single **Token Family**. When reuse is detected, *every* token in that family is killed at once.

```
Login Session #42 → Family ID: fam_42
  ├── RT_AAA (ROTATED)
  ├── RT_BBB (ROTATED) ← reuse detected here!
  ├── RT_CCC (ACTIVE)  ← could be in attacker's or user's hands
  └── (any future RTs in this chain)

→ Mark fam_42 entirely REVOKED
→ Force re-login for everyone holding any token from this family
```

> [!note] Why kill the whole family
> At the moment reuse is detected, you *cannot tell which party is the attacker*. The safe assumption is "one of them is hostile", and the right response is to distrust the entire chain. The legitimate user pays a tiny cost (re-login); the attacker is fully cut off.

### What the server must store

A [[JWT]] is self-contained, so verifying it normally needs no server state — but rotation and reuse detection *require* server-side state. Typically held in [[Redis]] or an RDB:

| Column | Meaning |
|--------|---------|
| `jti` | Unique ID of this Refresh Token (JWT `jti` claim) |
| `family_id` | The Token Family this RT belongs to |
| `user_id` | Which user it was issued to |
| `parent_jti` | Which RT it was rotated from (chain traversal) |
| `state` | `ACTIVE` / `ROTATED` / `REVOKED` |
| `issued_at`, `expires_at` | Issue / expiry timestamps |
| `replaced_by_jti` | The next RT in the chain after rotation |

Validation pseudocode:

```python
def exchange_refresh_token(rt):
    record = store.get(rt.jti)

    if record is None:
        raise InvalidToken()

    if record.state == "ROTATED":
        # A previously-used RT is back → suspected theft
        store.revoke_family(record.family_id)
        audit_log("RT reuse detected", family=record.family_id)
        raise SecurityIncident()

    if record.state == "REVOKED":
        raise RevokedToken()

    # Normal path
    new_at = issue_access_token(record.user_id)
    new_rt = issue_refresh_token(
        user_id=record.user_id,
        family_id=record.family_id,
        parent_jti=record.jti,
    )
    record.state = "ROTATED"
    record.replaced_by_jti = new_rt.jti
    store.save(record)
    return new_at, new_rt
```

> [!tip] Idempotency and the network-retry trap
> If a client sends `/token`, the response is dropped, and it retries with the same RT, that legitimate retry will look like reuse. Real-world implementations soften this with a short grace window (re-presenting the most recently used RT within N seconds returns the same response), an idempotency key, or client-side locking so concurrent refreshes don't fire.

### Standards basis: [[OAuth 2.1]] and the Security BCP

Refresh Token Rotation and reuse detection are codified in [[OAuth 2.1]] and the [OAuth 2.0 Security Best Current Practice (draft-ietf-oauth-security-topics)](https://datatracker.ietf.org/doc/draft-ietf-oauth-security-topics/) — they are **effectively mandatory for public clients** (SPAs, mobile apps), not optional.

### How it sits with the other defence axes

| Axis | What it blocks | Limitation |
|------|----------------|-----------|
| Short Access Token lifetime | Caps AT-theft blast radius | Doesn't help against RT theft |
| HttpOnly Cookie / HTTPS | Prevents theft itself | Can't stop XSS bypass / backend leaks |
| **Rotation + Reuse Detection** | **Detects and contains RT theft after the fact** | Doesn't stop the first misuse |
| [[DPoP]] / [[mTLS]] | Key-binds the token so theft alone is useless | More complex to deploy |

Rotation is *detection and containment*, not prevention. It only matters as part of a layered defence with the other axes.

> [!note] One-line summary
> Refresh Token Rotation = "Make every RT single-use, then treat any reappearance of a retired RT as a theft signal and revoke its entire family immediately" — an active, detection-driven defence against long-lived token theft.

### JWKS and kid: Public Key Distribution and Key Rotation

The last piece of the OIDC verification puzzle is "**where** and **how** does the app get the OP's public key?" The standard answer is [[JWKS]], and the identifier that lets the app pick the right key out of many is [[kid]].

### 1. What is JWKS — A Public Key Catalog

**JWKS (JSON Web Key Set)** = "a list of public keys for signature verification, published in JSON form."

In asymmetric signatures ([[RS256]] / [[ES256]]), the [[OpenID Provider]] **signs with a private key** and the [[Relying Party]] (the app) **verifies with a public key**. So where does the app get that public key?

- ❌ Hardcode it → the moment the OP rotates the key, every app on earth breaks.
- ❌ Manual config file → operators must update on every rotation.
- ✅ OP **publishes it as JSON at a standard URL**, and apps **fetch and cache it over HTTP** → JWKS.

```
                 ┌─────────────────────────────────┐
   Google OP     │  https://www.googleapis.com/    │
   (signer)      │     oauth2/v3/certs             │  ← jwks_uri
                 │  { "keys": [ {kid:..., n:..., e:...}, ... ] }
                 └─────────────────────────────────┘
                          ▲ anyone can HTTP GET
                          │ (public key, so no secret)
                          │
   My App ──── fetch ─────┘
   (verifier)
```

> [!note] JWK vs JWKS
> - **JWK** (JSON Web Key) = a **single** key represented as JSON.
> - **JWKS** (JSON Web Key **Set**) = an **array** of JWKs, wrapped under a single `keys` field.

### 2. What JWKS Actually Looks Like

Fetching the JWKS from the [[Google]] OP gives you roughly this JSON:

```json
{
  "keys": [
    {
      "kid": "abc123def456",
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "n": "0vx7agoebGcQSuuPiLJXZptN9nndrQmbXEps2aiAFbWhM78L...",
      "e": "AQAB"
    },
    {
      "kid": "xyz789ghi012",
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "n": "qjQ5UfdNZQYrkP6QSj3FjUL7L8DDcXkR9k...",
      "e": "AQAB"
    }
  ]
}
```

Field meanings:

| Field | Meaning |
|-------|---------|
| `kid` | **Key ID** — the identifier for this key. Matched against the token header's `kid` |
| `kty` | Key Type — `RSA`, `EC`, etc. |
| `use` | Purpose — `sig` (signature verification) / `enc` (encryption). OIDC uses `sig` |
| `alg` | Signature algorithm to use with this key (`RS256`, etc.) |
| `n` | RSA modulus (Base64URL) — the core of the public key |
| `e` | RSA exponent (usually `AQAB` = 65537) |

> [!tip] What n and e Really Are
> An RSA public key is mathematically the pair `(n, e)`. JWKS doesn't use PEM or X.509 — it just **embeds those two numbers directly in the JSON**. The library takes them and reconstructs an RSA Public Key object internally to verify with.

### 3. kid — The Name Tag That Says "I Was Signed by This Key"

[[OpenID Provider]]s typically **publish multiple public keys at once** — because of **key rotation**, new and old keys coexist for a while.

So how does the app know which key to verify with? → It matches against the **`kid` embedded in the token's Header**.

```
┌─────────────────────────────────────────────────────────────┐
│  ID Token (JWT)                                             │
│                                                             │
│  Header (decoded):                                          │
│  ┌────────────────────────────────────┐                     │
│  │ {                                  │                     │
│  │   "alg": "RS256",                  │                     │
│  │   "kid": "abc123def456" ───────────┼──┐                  │
│  │ }                                  │  │                  │
│  └────────────────────────────────────┘  │                  │
└──────────────────────────────────────────┼──────────────────┘
                                           │ match
                                           ▼
┌─────────────────────────────────────────────────────────────┐
│  JWKS                                                       │
│  {                                                          │
│    "keys": [                                                │
│      { "kid": "abc123def456", "n": "...", "e": "AQAB" },  ←✓│
│      { "kid": "xyz789ghi012", "n": "...", "e": "AQAB" }    │
│    ]                                                        │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘
```

> [!example] kid Matching Pseudocode
> ```python
> header = decode_header(id_token)
> kid    = header["kid"]           # "abc123def456"
> jwks   = fetch_jwks(jwks_uri)    # cache first
> 
> key = next(k for k in jwks["keys"] if k["kid"] == kid)
> if not key:                       # unknown kid → possibly rotated
>     jwks = fetch_jwks(jwks_uri, force_refresh=True)
>     key  = next(k for k in jwks["keys"] if k["kid"] == kid)
> 
> public_key = build_rsa_key(key["n"], key["e"])
> verify_signature(id_token, public_key, alg=key["alg"])
> ```

### 4. Key Rotation — Automatic Handling Without Code Changes

**Rotation scenario**: Say Google replaces its signing private key every six months as a security policy.

#### Before (hardcoded keys)

```
Day 1   : App has PUBLIC_KEY_v1 baked in → verification works
Day 180 : Google rotates → new tokens signed with PRIVATE_KEY_v2
Day 181 : App tries to verify with PUBLIC_KEY_v1 → ❌ FAIL
          → logins break globally → emergency deploy
```

#### After (JWKS + kid)

```
Day 1   : JWKS = [ {kid: "v1", ...} ]
          token header.kid = "v1"
          app finds "v1" in JWKS → verifies ✓

Day 175 : Google adds a new key (overlap period)
          JWKS = [ {kid: "v1", ...}, {kid: "v2", ...} ]
          Existing tokens (kid: v1) still verify ✓

Day 180 : Google starts signing new tokens with v2
          App receives token with header.kid = "v2"
          → not in cache → re-fetch JWKS → verify with v2 ✓

Day 200 : Google removes v1 from JWKS
          (all tokens signed with v1 have already expired)
          JWKS = [ {kid: "v2", ...} ]
```

> [!note] Why Rotation Is Zero-Downtime
> The new key isn't swapped in **instantly and alone** — it's **published alongside the old key**, then the old one is removed slowly. This **Overlap Strategy** lets the app transition naturally without any code change. The whole thing works because each token carries its own `kid`.

### 5. The App's Caching Strategy — How Should JWKS Be Fetched?

What if you **fetch JWKS on every request**?

- HTTP request to the OP per API call → **OP overload**, slower app
- Auth breaks entirely if the network blips
- Places like Google will rate-limit you

So **caching is essential**. But naive caching breaks on key rotation when a new kid arrives. The standard playbook is:

| Strategy | Behavior |
|----------|----------|
| **TTL-based cache** | Cache JWKS for 10 min–1 hr. Respect `Cache-Control` headers |
| **Force refresh on kid miss** | If the token's kid isn't in the cache, immediately re-fetch JWKS |
| **Rate limit** | Cap the kid-miss refresh frequency (e.g., 1 forced refresh per 10 min) |
| **Background refresh** | Optionally refresh in the background before TTL expires |

> [!warning] Watch Out for Infinite Fetch on kid Miss
> An attacker could send tokens with **made-up kids** that don't exist. A naive implementation would re-fetch JWKS every time → an **amplification attack** against the OP. Libraries typically expose options like `jwksRequestsPerMinute` to cap the fetch rate.

### 6. Library Behavior — Using `jwks-rsa` as an Example

In [[Node.js]], the most common library is [[jwks-rsa]], which implements the strategy above almost verbatim:

```javascript
import jwksClient from 'jwks-rsa';
import jwt from 'jsonwebtoken';

const client = jwksClient({
  jwksUri: 'https://www.googleapis.com/oauth2/v3/certs',
  cache: true,                  // in-memory caching
  cacheMaxEntries: 5,           // cache up to 5 keys
  cacheMaxAge: 10 * 60 * 1000,  // 10-minute TTL
  rateLimit: true,
  jwksRequestsPerMinute: 10,    // at most 10 fetches per minute
});

function getKey(header, callback) {
  // Library uses header.kid to fetch the matching public key
  client.getSigningKey(header.kid, (err, key) => {
    callback(err, key.getPublicKey());
  });
}

jwt.verify(idToken, getKey, {
  algorithms: ['RS256'],        // ← alg whitelist blocks alg:none and HS256 confusion
  issuer: 'https://accounts.google.com',
  audience: MY_CLIENT_ID,
}, (err, payload) => {
  if (err) return reject(err);
  resolve(payload);
});
```

> [!example] How the Flow Plays Out
> 1. `jwt.verify` extracts `kid` from the token header
> 2. Passes the header to the `getKey` callback
> 3. `jwks-rsa` searches its cache for the public key matching that `kid`
> 4. On cache miss, it HTTP GETs `jwksUri`, parses JWKS, and updates the cache
> 5. Builds an RSA Public Key object from `n` and `e` of the matched key
> 6. `jsonwebtoken` verifies the signature with that key and checks iss/aud/exp

The same job is done by `PyJWT` + `PyJWKClient` in [[Python]], `nimbus-jose-jwt`'s `RemoteJWKSet` in [[Java]], and `lestrrat-go/jwx`'s `jwk.Cache` in [[Go]].

### 7. One-Line Summary

> [!tip] JWKS & kid in One Line
> **JWKS** is the OP's published **catalog of public keys for signature verification**, and **kid** is the identifier that **matches a token to a key by name tag**. Together they let the OP rotate keys freely while apps follow along automatically — metaphorically, **JWKS = the key locker, kid = the name tag on each key**.

### OIDC Discovery: /.well-known/openid-configuration

[[OIDC Discovery]] is the mechanism by which **an OP (OpenID Provider) publishes all of its own metadata as JSON at a standardized URL**. With just the `issuer` value in hand, a client (RP) can auto-fetch the endpoints, key locations, and supported algorithms it needs to wire up OIDC. It is the **end of hardcoding** — a service manual maintained by the OP itself.

### Why it exists — the tragedy of hardcoding

To integrate [[OIDC]] manually, an app has to know a long list of things:

```
authorization_endpoint  →  where to send the user to log in
token_endpoint          →  where to exchange a code for tokens
userinfo_endpoint       →  where to fetch extra profile claims
jwks_uri                →  where to fetch the signing public keys
issuer                  →  what the ID Token's iss claim must equal
supported algorithms    →  RS256? ES256? PKCE methods? which scopes?
```

If you bake all of that into your code for every OP (Google, Apple, Auth0, Keycloak, Okta, …):

- Any URL tweak on the OP's side breaks every RP in the world.
- Routine operational changes — key rotation, adding an algorithm — require a code deploy each time.
- A single typo becomes a silent outage.

> [!note] The essence of Discovery
> Discovery is the OP saying out loud, in standardized JSON, **"this is what I'm called and this is how to use me."** The RP just listens.

### Location: /.well-known/openid-configuration

The standard path follows the `.well-known/` convention defined by [[RFC 5785]]. OIDC mandates the suffix `openid-configuration` ([[OIDC Discovery 1.0 spec]]).

```
https://{issuer}/.well-known/openid-configuration
```

Real examples:

| OP        | Discovery URL                                                          |
|-----------|------------------------------------------------------------------------|
| Google    | `https://accounts.google.com/.well-known/openid-configuration`         |
| Microsoft | `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration` |
| Auth0     | `https://{your-tenant}.auth0.com/.well-known/openid-configuration`     |
| Keycloak  | `https://{host}/realms/{realm}/.well-known/openid-configuration`       |

> [!tip] You start with just the issuer
> The only thing a user has to tell the RP is the `issuer` (or a domain from which it can be derived). Append `/.well-known/openid-configuration` and everything else falls out.

### Response shape — real JSON

Boiled down to the essentials, Google's real response looks like this:

```json
{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "revocation_endpoint": "https://oauth2.googleapis.com/revoke",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",

  "response_types_supported": ["code", "token", "id_token", "code token", "code id_token", "..."],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "email", "profile"],
  "token_endpoint_auth_methods_supported": ["client_secret_post", "client_secret_basic"],
  "claims_supported": ["aud", "email", "email_verified", "exp", "family_name", "given_name", "iat", "iss", "locale", "name", "picture", "sub"],
  "code_challenge_methods_supported": ["plain", "S256"],
  "grant_types_supported": ["authorization_code", "refresh_token", "..."]
}
```

What each field means:

| Field                                        | Meaning                                                                                  |
|----------------------------------------------|------------------------------------------------------------------------------------------|
| `issuer`                                     | The OP's unique identifier. The [[ID Token]]'s `iss` claim **must match this exactly**.  |
| `authorization_endpoint`                     | Where to redirect the user to log in / consent (the start of [[Authorization Code Flow]]) |
| `token_endpoint`                             | Where to exchange `code → access_token + id_token`                                       |
| `userinfo_endpoint`                          | Where to call with an Access Token for additional profile claims                         |
| `jwks_uri`                                   | Where the [[JWKS]] (signing-verification public keys) lives                              |
| `revocation_endpoint`                        | Token revocation endpoint                                                                |
| `id_token_signing_alg_values_supported`      | Algorithms the OP may use to sign ID Tokens (`RS256`, `ES256`, … — basis for your whitelist) |
| `scopes_supported` / `claims_supported`      | Requestable scopes and the claims that may be issued                                     |
| `code_challenge_methods_supported`           | [[PKCE]] challenge methods (`S256` recommended)                                          |
| `token_endpoint_auth_methods_supported`      | How the client authenticates itself to the token endpoint                                |
| `response_types_supported` / `grant_types_supported` | Which flows the OP supports                                                      |

> [!warning] issuer comparison is byte-for-byte
> The Discovery doc's `issuer` and the received [[ID Token]]'s `iss` claim must match **character for character**. A trailing slash difference is a verification failure. Multi-tenant OPs encode the tenant in `issuer`, which makes strict comparison even more critical.

### Discovery → JWKS → kid → verify, the whole picture

Discovery isn't the destination — it's the **entry point of the whole OIDC verification pipeline**. Here's how a single login unfolds from this metadata.

```
┌──────────────────────────────────────────────────────────────────────┐
│  Bootstrap: once on app start or first request                       │
│                                                                      │
│  1. You know the issuer ("https://accounts.google.com")              │
│  2. GET https://accounts.google.com/.well-known/openid-configuration │
│     → authorization_endpoint, token_endpoint, jwks_uri, ...          │
│  3. GET {jwks_uri}                                                   │
│     → JWKS = { keys: [ {kid: "abc", n, e, ...}, {kid: "xyz", ...} ] }│
│  4. Cache Discovery doc and JWKS in memory (respect Cache-Control)   │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Login: Authorization Code + PKCE                                    │
│                                                                      │
│  5. Redirect the user to authorization_endpoint                      │
│     (client_id, redirect_uri, scope=openid, state, nonce, PKCE)      │
│  6. OP redirects back to redirect_uri with ?code=...                 │
│  7. POST to token_endpoint to exchange the code                      │
│     → { access_token, id_token, refresh_token? }                     │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ID Token verification                                               │
│                                                                      │
│  8. id_token.header → { alg: "RS256", kid: "abc" }                    │
│  9. Pick key with kid="abc" from cached JWKS                          │
│       (missing? → refetch JWKS → still missing? → reject)            │
│ 10. RSA-Verify(signing_input, signature, that public key)            │
│ 11. Claims check:                                                    │
│       iss === Discovery.issuer                                       │
│       aud === client_id                                              │
│       exp > now                                                      │
│       nonce === (the nonce you stored)                               │
│       alg ∈ your whitelist (subset of id_token_signing_alg_values_supported) │
│ 12. All pass → login succeeds                                        │
└──────────────────────────────────────────────────────────────────────┘
```

> [!example] One analogy that ties it together
> - [[Discovery]] = the OP's **service manual / info desk**
> - `jwks_uri` = the **address of the key cabinet**
> - [[JWKS]] = the **key cabinet** itself
> - [[kid]] = the **name tag** on each key
> - The [[ID Token]]'s `header.kid` = a **note saying "I was sealed with key abc123"**
>
> The app walks up to the info desk, gets the key cabinet's address, opens the cabinet, pulls out the key whose tag matches the note, and uses it to verify the seal (signature).

### In practice — don't implement this by hand

Almost every language has an [[OIDC]] client library that handles Discovery for you, so real code is usually this short:

```ts
// Node.js (openid-client) example
import { Issuer } from 'openid-client'

const issuer = await Issuer.discover('https://accounts.google.com')
// → internally GETs /.well-known/openid-configuration
//   and fills issuer.metadata with every endpoint

const client = new issuer.Client({
  client_id: process.env.CLIENT_ID,
  client_secret: process.env.CLIENT_SECRET,
  redirect_uris: ['https://app.example.com/callback'],
  response_types: ['code'],
})

// In the login callback:
const tokens = await client.callback(redirect_uri, params, { nonce, state, code_verifier })
//   → token exchange + ID Token sig / iss / aud / exp / nonce verification, all automatic
//   → fetches, caches, and rotates jwks_uri keys under the hood
```

Helpers like [[jwks-rsa]] take just the `jwks_uri` and handle caching, key rotation, and rate-limit pressure for you.

### Caching, key rotation, failure modes

> [!warning] Don't fetch Discovery/JWKS on every request
> Discovery and JWKS rarely change. If you stampede the OP it will rate-limit or block you. The standard pattern is **cache in memory**, honor `Cache-Control: max-age`, and **refetch JWKS only when a token's `kid` is missing from the cache**.

A typical caching policy:

| Resource             | Cache TTL              | Invalidation trigger                                       |
|---------------------|------------------------|-----------------------------------------------------------|
| Discovery document  | hours → 24h            | TTL expiry, explicit restart                               |
| JWKS                | hours                  | A token arrives with an unknown `kid` → force refetch     |
| Negative cache (unknown kid) | short (minutes)| DoS guard — prevents a fetch storm when many bogus kids arrive |

### Security checklist

> [!warning] Before you trust a Discovery doc
> - **HTTPS only**: `.well-known/openid-configuration` must be fetched over HTTPS. Over plaintext HTTP, a MITM can swap `jwks_uri` for an attacker-controlled key.
> - **Whitelist allowed issuers**: even multi-tenant apps must keep an explicit list of acceptable issuers — anyone can host a fake Discovery doc.
> - **Whitelist algorithms in code**: don't blindly trust `id_token_signing_alg_values_supported`. **Pin the expected algorithm in your verifier code.** This is your baseline defense against `alg:none` attacks and `RS256 → HS256` confusion attacks.
> - **Exact issuer match**: the Discovery `issuer` and the ID Token `iss` must match byte for byte.

### One-line takeaway

> [!note] Bottom line
> **Discovery is the standard metadata document that lets a client auto-discover every coordinate of an OIDC integration — endpoints, key locations, supported algorithms — from a single `issuer` value.** The pipeline `/.well-known/openid-configuration` → `jwks_uri` → JWKS → match the token's `kid` → verify the signature → verify the claims is the spine of OIDC trust.

## Diagram

[[canvas/OIDC와_OAuth_2.0_완벽_이해.canvas|Concept map]]
