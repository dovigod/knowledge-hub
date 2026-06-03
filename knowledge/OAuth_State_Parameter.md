---
id: 019e8cf7-8e7b-738c-aacf-41f8fecaf102
name: OAuth State Parameter
aliases:
  - oauth-state
  - state
  - state parameter
updated_at: '2026-06-03T10:11:34.651Z'
summary: >-
  An opaque, unguessable value the client includes in OAuth/OIDC authorization
  requests to bind the callback to the original session and defend against CSRF.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# OAuth State Parameter

## Overview
The `state` parameter is an opaque, client-generated value sent with an OAuth 2.0 / OIDC authorization request and echoed back on the redirect callback. It binds the callback to the originating browser session, defeating **CSRF** attacks where an attacker tricks a victim's browser into completing an OAuth flow against the victim's account.

## Notes
- Generated as a cryptographically random string (e.g., 128+ bits of entropy) per authorization attempt.
- Stored server-side or in a signed/HttpOnly cookie tied to the session before the redirect.
- On callback, the server compares the returned `state` to the stored value — any mismatch aborts the flow.
- Without `state`, an attacker could craft a malicious callback URL with their own authorization code, tricking the victim's logged-in browser into linking the attacker's account.
- Distinct from `nonce`: `state` protects the request/callback channel (CSRF); `nonce` protects the ID Token from replay/injection.

---

## 한국어

### 개요
`state` 파라미터는 클라이언트가 생성한 opaque 값으로, OAuth 2.0 / OIDC authorization 요청에 함께 보내고 redirect callback에서 그대로 돌려받습니다. callback을 최초 브라우저 세션과 묶어 **CSRF** 공격 — 공격자가 피해자의 브라우저를 속여 피해자 계정으로 OAuth flow를 완료시키는 공격 — 을 방어합니다.

### 노트
- authorization 시도마다 암호학적으로 무작위인 문자열(예: 128비트 이상의 엔트로피)로 생성합니다.
- redirect 전에 서버 사이드 또는 세션에 묶인 signed/HttpOnly 쿠키에 저장합니다.
- callback 시 서버는 반환된 `state`를 저장된 값과 비교하며, 불일치 시 flow를 중단합니다.
- `state`가 없으면 공격자가 자신의 authorization code가 담긴 악성 callback URL을 만들어, 로그인된 피해자의 브라우저가 공격자 계정을 연결하도록 속일 수 있습니다.
- `nonce`와 구분: `state`는 요청/callback 채널(CSRF)을 보호하고, `nonce`는 ID Token을 replay/injection 으로부터 보호합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
