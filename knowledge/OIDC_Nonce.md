---
id: 019e8cf7-9d4d-761a-88f3-40476b17c1a6
name: OIDC Nonce
aliases:
  - id-token nonce
  - nonce
  - oidc-nonce
updated_at: '2026-06-03T10:11:38.445Z'
summary: >-
  A client-generated random value bound to the ID Token's `nonce` claim,
  preventing token replay and ID-token injection attacks in OpenID Connect
  flows.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OIDC Nonce

## Overview
In OpenID Connect, `nonce` is a random value the client generates per authentication request and embeds in the authorization URL. The IdP copies it into the `nonce` claim of the issued **ID Token**. On receipt, the client verifies the claim matches the value it originally sent — defeating **replay** and **ID-token injection** attacks.

## Notes
- Required for the Implicit and Hybrid flows; recommended for Authorization Code flow.
- Stored client-side (session, signed cookie) before redirect; checked after the ID Token is validated.
- An attacker who somehow captures an old ID Token cannot reuse it: their fresh authorization request would carry a *new* nonce, but the stolen token still carries the *old* one.
- Without nonce, an attacker who intercepts a valid ID Token could inject it into another user's session and impersonate the original user.
- Different from `state`: `state` protects the redirect/callback channel; `nonce` protects the ID Token itself.

---

## 한국어

### 개요
OpenID Connect에서 `nonce`는 클라이언트가 인증 요청마다 생성해 authorization URL에 넣는 무작위 값입니다. IdP는 이 값을 발급된 **ID Token**의 `nonce` claim에 복사해 넣습니다. 수신 시 클라이언트는 claim 값이 자신이 보낸 값과 일치하는지 검증하여 **replay** 공격과 **ID-token injection** 공격을 방어합니다.

### 노트
- Implicit과 Hybrid flow에서는 필수이며, Authorization Code flow에서도 권장됩니다.
- redirect 전에 클라이언트 측(session, signed cookie)에 저장하고, ID Token 검증 후에 비교합니다.
- 공격자가 어떤 식으로든 과거 ID Token을 탈취하더라도 재사용할 수 없습니다 — 공격자의 새 authorization 요청에는 *새* nonce가 담기지만, 훔친 토큰에는 *옛* nonce가 들어있기 때문입니다.
- nonce가 없으면 유효한 ID Token을 가로챈 공격자가 다른 사용자의 세션에 주입해 원래 사용자를 가장할 수 있습니다.
- `state`와 구분: `state`는 redirect/callback 채널을 보호하고, `nonce`는 ID Token 자체를 보호합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
