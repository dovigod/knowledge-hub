---
id: 019e8cf7-7c8e-754c-9fbd-f1fda2b0ee56
name: OpenID Connect
aliases:
  - OIDC
  - OpenID
  - OpenID Connect
  - openid connect
  - openid-connect
updated_at: '2026-06-07T08:26:14.124Z'
summary: >-
  An identity layer built on top of OAuth 2.0 that lets clients verify the
  end-user's identity and obtain basic profile information via an ID Token.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OpenID Connect

## Overview
OpenID Connect (OIDC) is an authentication protocol that sits on top of [[OAuth 2.0]]. While OAuth 2.0 is about *authorization* (granting access to resources), OIDC adds *authentication* — proving who the user is — by introducing the **ID Token**, a signed [[JWT]] issued by the Identity Provider (IdP).

> [!note] OAuth vs OIDC in one line
> OAuth 2.0 answers "what can this client do?"; OIDC answers "who is the user?" via the ID Token.

## Notes
- Core flow: client redirects user to IdP → user authenticates → IdP returns an authorization code → client exchanges code for an **Access Token** + **ID Token** (+ optional Refresh Token).
- The **ID Token** is a [[JWT]] containing claims like `iss`, `sub`, `aud`, `exp`, `iat`, `nonce`, and user profile fields. The client validates its signature against the IdP's JWKS.
- The **UserInfo endpoint** returns additional profile claims when called with the access token.
- Standard scopes: `openid` (required), `profile`, `email`, `address`, `phone`.
- Common flows: Authorization Code (with [[PKCE]] for public clients), Implicit (legacy, discouraged), Hybrid.
- Major IdPs: Google, Microsoft Entra ID, Okta, Auth0, Keycloak.

## state and nonce
Two short-lived random values bound to a login attempt — they target *different* attacks and are not interchangeable.

> [!tip] `state` — CSRF defense on the redirect
> Client generates random `state`, stores it (cookie/session), sends it in the authorization request. IdP echoes it back on the callback. Client compares: if it doesn't match the stored value, the callback didn't originate from *this* user's login flow → reject. Without `state`, an attacker can trick a victim's browser into completing a callback the attacker initiated, linking the victim's session to the attacker's account (or vice versa).

> [!tip] `nonce` — replay/token-injection defense on the ID Token
> Client generates random `nonce`, sends it in the authorization request. IdP embeds it as a claim *inside* the issued ID Token. Client verifies `id_token.nonce` equals the value it originally sent. Because the nonce is **signed into the JWT**, an attacker can't reuse a stolen/replayed ID Token from another session — its embedded nonce won't match.

> [!warning] Don't conflate them
> `state` lives in the redirect URL (transport-layer integrity of the callback). `nonce` lives inside the signed token (integrity of the token-to-session binding). Skip either and the other won't cover the gap.

---

## 한국어

### 개요
OpenID Connect(OIDC)는 [[OAuth 2.0]] 위에 얹은 인증 프로토콜입니다. OAuth 2.0이 *인가(authorization)* — 리소스 접근 권한 부여 — 에 관한 것이라면, OIDC는 IdP(Identity Provider)가 발급하는 서명된 [[JWT]]인 **ID Token**을 도입해 *인증(authentication)* — 사용자가 누구인지 증명 — 기능을 추가합니다.

> [!note] OAuth와 OIDC, 한 줄 요약
> OAuth 2.0은 "이 클라이언트가 무엇을 할 수 있나?"를, OIDC는 ID Token을 통해 "이 사용자는 누구인가?"를 답합니다.

### 노트
- 핵심 흐름: 클라이언트가 사용자를 IdP로 리다이렉트 → 사용자 인증 → IdP가 authorization code 반환 → 클라이언트가 code를 **Access Token** + **ID Token** (+ 선택적 Refresh Token) 으로 교환.
- **ID Token**은 `iss`, `sub`, `aud`, `exp`, `iat`, `nonce` 및 사용자 프로필 필드 등의 claim을 포함하는 [[JWT]]이며, 클라이언트는 IdP의 JWKS로 서명을 검증합니다.
- **UserInfo 엔드포인트**는 access token으로 호출하면 추가 프로필 claim을 반환합니다.
- 표준 scope: `openid` (필수), `profile`, `email`, `address`, `phone`.
- 주요 flow: Authorization Code (public 클라이언트는 [[PKCE]] 동반), Implicit (레거시, 권장하지 않음), Hybrid.
- 대표 IdP: Google, Microsoft Entra ID, Okta, Auth0, Keycloak.

### state와 nonce
로그인 시도 한 건에 묶이는 두 개의 짧은 랜덤 값 — 서로 *다른* 공격을 막으며 대체재가 아닙니다.

> [!tip] `state` — 리다이렉트에 대한 CSRF 방어
> 클라이언트가 랜덤 `state`를 생성해 (쿠키/세션에) 저장한 뒤 authorization 요청에 실어 보냅니다. IdP는 콜백에서 이 값을 그대로 돌려줍니다. 클라이언트는 저장된 값과 비교해 일치하지 않으면 — 즉 *이* 사용자의 로그인 흐름에서 시작된 콜백이 아니라면 — 거부합니다. `state`가 없으면, 공격자가 시작한 콜백을 피해자의 브라우저로 완료하게 만들어 피해자 세션을 공격자 계정에 (또는 그 반대로) 연결해버릴 수 있습니다.

> [!tip] `nonce` — ID Token에 대한 재전송/토큰 주입 방어
> 클라이언트가 랜덤 `nonce`를 생성해 authorization 요청에 보냅니다. IdP는 발급하는 ID Token의 claim *내부*에 이 값을 박아 넣습니다. 클라이언트는 `id_token.nonce`가 원래 보냈던 값과 같은지 검증합니다. nonce가 **JWT 안에 서명되어 있기 때문에**, 공격자는 다른 세션에서 탈취·재사용한 ID Token을 갖다 붙일 수 없습니다 — 토큰 안에 박힌 nonce가 맞지 않습니다.

> [!warning] 둘을 혼동하지 말 것
> `state`는 리다이렉트 URL에 실립니다(콜백의 전송 계층 무결성). `nonce`는 서명된 토큰 안에 들어갑니다(토큰-세션 바인딩의 무결성). 둘 중 하나라도 빠뜨리면 나머지가 그 빈틈을 메워주지 않습니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
