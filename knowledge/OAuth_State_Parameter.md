---
id: 019e8cf7-8e7b-738c-aacf-41f8fecaf102
name: OAuth State Parameter
aliases:
  - OAuth state
  - oauth state
  - oauth-state
  - oidc state
  - state
  - state param
  - state parameter
updated_at: '2026-06-07T08:27:09.935Z'
summary: >-
  An opaque, unguessable value the client includes in OAuth/OIDC authorization
  requests to bind the callback to the original session and defend against CSRF.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OAuth State Parameter

## Overview
The `state` parameter is an opaque, client-generated value sent with an [[OAuth 2.0]] / [[OIDC]] authorization request and echoed back on the redirect callback. It binds the callback to the originating browser session, defeating **CSRF** attacks where an attacker tricks a victim's browser into completing an OAuth flow against the victim's account.

> [!note] How `state` actually does its job
> The client generates a random value, stores it tied to the *current browser session* before redirect, and on callback verifies the returned `state` matches that stored value. The match proves the callback belongs to the same browser that started the flow — no match, no continuation.

## Notes
- Generated as a cryptographically random string (e.g., 128+ bits of entropy) per authorization attempt.
- Stored server-side or in a signed/HttpOnly cookie tied to the session before the redirect.
- On callback, the server compares the returned `state` to the stored value — any mismatch aborts the flow.
- Without `state`, an attacker could craft a malicious callback URL with their own authorization code, tricking the victim's logged-in browser into linking the attacker's account.
- Distinct from [[Nonce]]: `state` protects the request/callback channel (CSRF); `nonce` protects the ID Token from replay/injection. Together they cover the two distinct attack surfaces of an [[OIDC]] flow.
- Server-side storage for `state` is often backed by a fast key-value store like [[Redis]] so the lookup on callback stays sub-millisecond and the entry can be auto-expired.

> [!warning] One-shot, single-use
> Treat the stored `state` like a CSRF token: consume it on the first successful callback and invalidate it. Reusing `state` across attempts re-opens the window the parameter was designed to close.

---

## 한국어

### 개요
`state` 파라미터는 클라이언트가 생성한 opaque 값으로, [[OAuth 2.0]] / [[OIDC]] authorization 요청에 함께 보내고 redirect callback에서 그대로 돌려받습니다. callback을 최초 브라우저 세션과 묶어 **CSRF** 공격 — 공격자가 피해자의 브라우저를 속여 피해자 계정으로 OAuth flow를 완료시키는 공격 — 을 방어합니다.

> [!note] `state`가 실제로 제 역할을 다하는 방식
> 클라이언트가 무작위 값을 생성해 redirect 직전에 *현재 브라우저 세션*에 묶어 저장하고, callback 시 돌려받은 `state`가 저장된 값과 일치하는지 확인합니다. 일치한다는 사실 자체가 "이 callback이 flow를 시작한 그 브라우저에서 왔다"는 증명이 됩니다 — 불일치면 진행을 중단합니다.

### 노트
- authorization 시도마다 암호학적으로 무작위인 문자열(예: 128비트 이상의 엔트로피)로 생성합니다.
- redirect 전에 서버 사이드 또는 세션에 묶인 signed/HttpOnly 쿠키에 저장합니다.
- callback 시 서버는 반환된 `state`를 저장된 값과 비교하며, 불일치 시 flow를 중단합니다.
- `state`가 없으면 공격자가 자신의 authorization code가 담긴 악성 callback URL을 만들어, 로그인된 피해자의 브라우저가 공격자 계정을 연결하도록 속일 수 있습니다.
- [[Nonce]]와 구분: `state`는 요청/callback 채널(CSRF)을 보호하고, `nonce`는 ID Token을 replay/injection 으로부터 보호합니다. 둘이 합쳐져야 [[OIDC]] flow의 서로 다른 두 공격면이 모두 막힙니다.
- `state`의 서버 사이드 저장소로는 [[Redis]] 같은 빠른 key-value store를 자주 씁니다 — callback에서의 조회를 sub-millisecond로 유지하고 자동 만료까지 걸 수 있기 때문입니다.

> [!warning] 일회성, 단일 사용
> 저장된 `state`는 CSRF 토큰처럼 다루세요: 첫 callback 성공 시 소비하고 무효화합니다. 여러 시도에 걸쳐 재사용하면, 이 파라미터가 막으려던 바로 그 공격 창이 다시 열립니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
