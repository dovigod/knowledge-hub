---
id: 019e8cf7-9d4d-761a-88f3-40476b17c1a6
name: OIDC Nonce
aliases:
  - OIDC nonce
  - auth nonce
  - id token nonce
  - id-token nonce
  - nonce
  - oidc nonce
  - oidc-nonce
updated_at: '2026-06-07T08:27:31.957Z'
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
In [[OpenID Connect]], `nonce` is a random value the client generates per authentication request and embeds in the authorization URL. The IdP copies it into the `nonce` claim of the issued **[[ID Token]]**. On receipt, the client verifies the claim matches the value it originally sent — defeating **replay** and **ID-token injection** attacks.

> [!tip] Mental model
> `nonce` binds an [[ID Token]] to the specific authentication request that asked for it. A token without a matching live nonce is "free-floating" and must be rejected.

## Notes
- Required for the [[Implicit Flow]] and [[Hybrid Flow]]; recommended for [[Authorization Code Flow]].
- Stored client-side ([[session]], [[signed cookie]]) before redirect; checked after the [[ID Token]] is validated.
- An attacker who somehow captures an old [[ID Token]] cannot reuse it: their fresh authorization request would carry a *new* nonce, but the stolen token still carries the *old* one.
- Without nonce, an attacker who intercepts a valid [[ID Token]] could inject it into another user's session and impersonate the original user.
- Different from [[state]]: `state` protects the redirect/callback channel (CSRF on the authorization response); `nonce` protects the [[ID Token]] itself (replay/injection on the token).
- Pairs naturally with [[state]] — together they cover both halves of the OIDC handshake: `state` guards the callback, `nonce` guards the token.

> [!warning] Common mistake
> Reusing the same `nonce` across requests, or skipping verification when the [[ID Token]] "looks valid" by signature alone, collapses the entire defense — signature validity proves the IdP issued it, not that *this* client requested it *now*.

---

## 한국어

### 개요
[[OpenID Connect]]에서 `nonce`는 클라이언트가 인증 요청마다 생성해 authorization URL에 넣는 무작위 값입니다. IdP는 이 값을 발급된 **[[ID Token]]**의 `nonce` claim에 복사해 넣습니다. 수신 시 클라이언트는 claim 값이 자신이 보낸 값과 일치하는지 검증하여 **replay** 공격과 **ID-token injection** 공격을 방어합니다.

> [!tip] 핵심 개념
> `nonce`는 [[ID Token]]을 그 토큰을 요청한 *바로 그* 인증 요청에 묶어줍니다. 살아있는 nonce와 일치하지 않는 토큰은 "떠다니는" 토큰이며 거부해야 합니다.

### 노트
- [[Implicit Flow]]와 [[Hybrid Flow]]에서는 필수이며, [[Authorization Code Flow]]에서도 권장됩니다.
- redirect 전에 클라이언트 측([[session]], [[signed cookie]])에 저장하고, [[ID Token]] 검증 후에 비교합니다.
- 공격자가 어떤 식으로든 과거 [[ID Token]]을 탈취하더라도 재사용할 수 없습니다 — 공격자의 새 authorization 요청에는 *새* nonce가 담기지만, 훔친 토큰에는 *옛* nonce가 들어있기 때문입니다.
- nonce가 없으면 유효한 [[ID Token]]을 가로챈 공격자가 다른 사용자의 세션에 주입해 원래 사용자를 가장할 수 있습니다.
- [[state]]와 구분: `state`는 redirect/callback 채널을 보호하고(authorization 응답에 대한 CSRF), `nonce`는 [[ID Token]] 자체를 보호합니다(토큰에 대한 replay/injection).
- [[state]]와 자연스럽게 짝을 이룹니다 — 둘이 함께 OIDC 핸드셰이크의 양쪽을 모두 보호합니다: `state`는 callback을, `nonce`는 token을 지킵니다.

> [!warning] 자주 하는 실수
> 여러 요청에서 같은 `nonce`를 재사용하거나, 서명만으로 [[ID Token]]이 "유효해 보인다"고 검증을 건너뛰면 방어 전체가 무너집니다 — 서명 유효성은 IdP가 발급했다는 사실만 증명할 뿐, *이 클라이언트*가 *지금* 요청했다는 사실은 증명하지 못합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
