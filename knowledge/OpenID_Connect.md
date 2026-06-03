---
id: 019e8cf7-7c8e-754c-9fbd-f1fda2b0ee56
name: OpenID Connect
aliases:
  - OIDC
  - OpenID
  - openid-connect
updated_at: '2026-06-03T10:11:30.062Z'
summary: >-
  An identity layer built on top of OAuth 2.0 that lets clients verify the
  end-user's identity and obtain basic profile information via an ID Token.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OpenID Connect

## Overview
OpenID Connect (OIDC) is an authentication protocol that sits on top of OAuth 2.0. While OAuth 2.0 is about *authorization* (granting access to resources), OIDC adds *authentication* — proving who the user is — by introducing the **ID Token**, a signed JWT issued by the Identity Provider (IdP).

## Notes
- Core flow: client redirects user to IdP → user authenticates → IdP returns an authorization code → client exchanges code for an **Access Token** + **ID Token** (+ optional Refresh Token).
- The **ID Token** is a JWT containing claims like `iss`, `sub`, `aud`, `exp`, `iat`, `nonce`, and user profile fields. The client validates its signature against the IdP's JWKS.
- The **UserInfo endpoint** returns additional profile claims when called with the access token.
- Standard scopes: `openid` (required), `profile`, `email`, `address`, `phone`.
- Common flows: Authorization Code (with PKCE for public clients), Implicit (legacy, discouraged), Hybrid.
- `state` and `nonce` parameters defend against CSRF and replay/token-injection attacks respectively.
- Major IdPs: Google, Microsoft Entra ID, Okta, Auth0, Keycloak.

---

## 한국어

### 개요
OpenID Connect(OIDC)는 OAuth 2.0 위에 얹은 인증 프로토콜입니다. OAuth 2.0이 *인가(authorization)* — 리소스 접근 권한 부여 — 에 관한 것이라면, OIDC는 IdP(Identity Provider)가 발급하는 서명된 JWT인 **ID Token**을 도입해 *인증(authentication)* — 사용자가 누구인지 증명 — 기능을 추가합니다.

### 노트
- 핵심 흐름: 클라이언트가 사용자를 IdP로 리다이렉트 → 사용자 인증 → IdP가 authorization code 반환 → 클라이언트가 code를 **Access Token** + **ID Token** (+ 선택적 Refresh Token) 으로 교환.
- **ID Token**은 `iss`, `sub`, `aud`, `exp`, `iat`, `nonce` 및 사용자 프로필 필드 등의 claim을 포함하는 JWT이며, 클라이언트는 IdP의 JWKS로 서명을 검증합니다.
- **UserInfo 엔드포인트**는 access token으로 호출하면 추가 프로필 claim을 반환합니다.
- 표준 scope: `openid` (필수), `profile`, `email`, `address`, `phone`.
- 주요 flow: Authorization Code (public 클라이언트는 PKCE 동반), Implicit (레거시, 권장하지 않음), Hybrid.
- `state`와 `nonce` 파라미터는 각각 CSRF와 replay/token-injection 공격을 방어합니다.
- 대표 IdP: Google, Microsoft Entra ID, Okta, Auth0, Keycloak.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
