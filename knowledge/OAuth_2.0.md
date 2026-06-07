---
id: 019e8cf7-7c94-7497-a57f-9c1d30f127a4
name: OAuth 2.0
aliases:
  - OAuth
  - OAuth 2.0
  - OAuth2
  - oauth
  - oauth 2
  - oauth-2.0
updated_at: '2026-06-07T08:26:43.293Z'
summary: >-
  An authorization framework that lets a third-party application obtain limited
  access to a user's resources on an HTTP service without exposing the user's
  credentials.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OAuth 2.0

## Overview
OAuth 2.0 is an authorization framework (RFC 6749) that delegates limited access to protected resources from a Resource Owner to a Client, mediated by an Authorization Server that issues Access Tokens. It explicitly does **not** define authentication — that gap is what [[OIDC]] fills.

> [!warning] OAuth ≠ Authentication
> Using an access token to identify a user is unsafe. Access tokens prove *authorization* to a resource, not *who* the bearer is. For identity, use [[OIDC]]'s ID Token.

## Notes
- Four roles: Resource Owner (user), Client (app), Authorization Server (issues tokens), Resource Server (hosts data).
- Grant types: Authorization Code (with PKCE), Client Credentials (machine-to-machine), Refresh Token, Device Code, Resource Owner Password (deprecated), Implicit (deprecated).
- Access tokens are typically opaque or JWT bearer tokens scoped to specific resources.
- PKCE (Proof Key for Code Exchange) is now mandatory for all clients, not just public ones (OAuth 2.1).
- The `state` parameter is a CSRF defense in the Authorization Code flow: the Client generates an unguessable value, binds it to the user's session (often via [[Redis]] or signed cookie), and verifies the Authorization Server echoes it back on the redirect. Mismatch ⇒ reject the callback.
- `nonce` is an [[OIDC]] concept (not OAuth 2.0 proper) for replay protection on the ID Token — included in the request, embedded in the issued ID Token, verified on receipt.

> [!tip] state vs nonce — how each fulfills its role
> - **`state`** works because it's *bound to the user's browser session before redirect* and *verified after redirect*. An attacker forging a callback URL cannot guess the value, and a stale/replayed callback won't match the session's stored value. Storage is typically server-side ([[Redis]]) or in a signed cookie — both give the Client a trusted reference to compare against.
> - **`nonce`** works because it's *embedded inside the signed ID Token* by the Authorization Server. A replayed ID Token from an old auth request will carry an old nonce that no longer matches the Client's current expectation, so the token is rejected even though its signature is valid.
> Both must be unpredictable and single-use.

---

## 한국어

### 개요
OAuth 2.0은 인가 프레임워크(RFC 6749)로, Authorization Server가 발급하는 Access Token을 매개로 Resource Owner의 보호된 리소스에 대한 제한된 접근 권한을 Client에게 위임합니다. 명시적으로 인증(authentication)은 정의하지 **않으며**, 이 공백을 [[OIDC]]가 채웁니다.

> [!warning] OAuth ≠ 인증
> Access token으로 사용자를 식별하는 것은 안전하지 않습니다. Access token은 리소스에 대한 *인가*를 증명할 뿐, *누구인지*는 증명하지 않습니다. 신원이 필요하다면 [[OIDC]]의 ID Token을 사용하세요.

### 노트
- 4개 역할: Resource Owner (사용자), Client (앱), Authorization Server (토큰 발급), Resource Server (데이터 보관).
- Grant 타입: Authorization Code (PKCE 동반), Client Credentials (서버 간), Refresh Token, Device Code, Resource Owner Password (deprecated), Implicit (deprecated).
- Access token은 일반적으로 opaque 토큰 또는 특정 리소스에 scope이 지정된 JWT bearer 토큰입니다.
- PKCE(Proof Key for Code Exchange)는 OAuth 2.1에서 public 클라이언트뿐 아니라 모든 클라이언트에 필수입니다.
- `state` 파라미터는 Authorization Code 플로우의 CSRF 방어 장치입니다: Client가 예측 불가능한 값을 생성해 사용자 세션에 바인딩(보통 [[Redis]] 또는 서명된 쿠키)하고, Authorization Server가 리다이렉트로 돌려준 값과 일치하는지 검증합니다. 불일치면 콜백을 거부합니다.
- `nonce`는 [[OIDC]] 개념(OAuth 2.0 자체는 아님)으로 ID Token의 재전송 공격을 막습니다 — 요청에 포함하고, 발급된 ID Token에 박혀서 돌아오면 검증합니다.

> [!tip] state와 nonce — 각각이 어떻게 역할을 다하는가
> - **`state`**가 작동하는 이유: 리다이렉트 *전에* 사용자 브라우저 세션에 바인딩되고, 리다이렉트 *후에* 검증되기 때문입니다. 공격자가 콜백 URL을 위조해도 값을 추측할 수 없고, 오래된/재전송된 콜백은 세션에 저장된 값과 일치하지 않습니다. 저장은 보통 서버 측([[Redis]]) 또는 서명된 쿠키로 — 둘 다 Client가 비교할 수 있는 신뢰 가능한 기준을 제공합니다.
> - **`nonce`**가 작동하는 이유: Authorization Server가 서명한 ID Token *내부에* 박혀 있기 때문입니다. 옛 인증 요청에서 재전송된 ID Token은 오래된 nonce를 갖고 있어 Client의 현재 기대값과 일치하지 않으므로, 서명이 유효하더라도 거부됩니다.
> 둘 다 예측 불가능하고 일회용이어야 합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
