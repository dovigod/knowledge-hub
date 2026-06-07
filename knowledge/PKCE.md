---
id: 019ea131-3f4d-73f7-bb82-2dd8cb9f209f
name: PKCE
aliases:
  - Proof Key for Code Exchange
  - oauth pkce
  - pkce
updated_at: '2026-06-07T08:26:59.789Z'
summary: >-
  An OAuth 2.0 extension (RFC 7636) that binds the authorization code to the
  original client via a code_verifier/code_challenge pair to prevent code
  interception.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# PKCE

## Overview

PKCE ("pixy", Proof Key for Code Exchange, RFC 7636) is an extension to the [[OAuth 2.0]] Authorization Code Flow that prevents an attacker who intercepts the authorization code from exchanging it for tokens. It is mandatory in modern [[OpenID Connect]] best practice for public clients (SPA, mobile, native).

> [!note] How it works in one sentence
> The client proves at the token endpoint that it is the same party that started the authorization request, by revealing a secret (`code_verifier`) whose hash (`code_challenge`) it committed to up front.

## Notes

- **Flow**:
  1. Client generates random `code_verifier` (43–128 chars).
  2. Computes `code_challenge = BASE64URL(SHA256(code_verifier))`.
  3. Sends `code_challenge` + `code_challenge_method=S256` in `/authorize`.
  4. After receiving the authorization code, sends `code_verifier` in `/token`.
  5. Auth server hashes `code_verifier`, compares to stored `code_challenge`.
- **Why it matters**: public clients can't safely hold a client secret; PKCE replaces that secret with a per-request proof.
- Use `S256` (SHA-256); the `plain` method exists only for constrained clients and is discouraged.
- Complements [[state]] (CSRF) and [[nonce]] (ID token replay) — they protect different legs of the flow.

> [!tip] Use PKCE everywhere
> RFC 9700 (OAuth 2.0 Security BCP) recommends PKCE even for confidential clients now.

---

## 한국어

### 개요

PKCE("픽시", Proof Key for Code Exchange, RFC 7636)는 [[OAuth 2.0]] Authorization Code Flow의 확장 규격으로, authorization code를 가로챈 공격자가 토큰으로 교환하지 못하도록 막는다. 현대 [[OpenID Connect]] 베스트 프랙티스에서는 public client(SPA, 모바일, 네이티브)에 필수.

> [!note] 핵심 동작 한 줄 요약
> 클라이언트는 인증 요청 시 비밀값(`code_verifier`)의 해시(`code_challenge`)를 미리 등록해 두고, 토큰 교환 시 원본 `code_verifier`를 공개해 "내가 그 요청을 시작한 그 당사자"임을 증명한다.

### 노트

- **흐름**:
  1. 클라이언트가 난수 `code_verifier`(43~128자) 생성.
  2. `code_challenge = BASE64URL(SHA256(code_verifier))` 계산.
  3. `/authorize`에 `code_challenge` + `code_challenge_method=S256` 전달.
  4. authorization code 수신 후 `/token`에서 `code_verifier` 전달.
  5. Auth server가 `code_verifier`를 해시해 저장된 `code_challenge`와 비교.
- **중요한 이유**: public client는 client secret을 안전하게 보관할 수 없음. PKCE가 그 비밀을 요청별 증명으로 대체.
- `S256`(SHA-256) 사용. `plain` 방식은 제약 환경 전용이며 권장되지 않음.
- [[state]](CSRF 방어), [[nonce]](ID token replay 방어)와 **보완 관계** — 보호 구간이 서로 다름.

> [!tip] PKCE는 어디서나 권장
> RFC 9700(OAuth 2.0 Security BCP)은 이제 confidential client에도 PKCE 적용을 권고한다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
