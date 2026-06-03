---
id: 019e8cf8-0979-7586-8dcb-57edb4cb4b32
name: JSON Web Token
aliases:
  - JWT
  - json-web-token
  - jwt-token
updated_at: '2026-06-03T10:12:06.137Z'
summary: >-
  A compact, URL-safe, signed (and optionally encrypted) token format encoding
  claims as a base64-url payload, widely used for authentication and
  authorization.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# JSON Web Token

## Overview
A JSON Web Token (JWT, RFC 7519) is a compact token consisting of three base64url-encoded parts — header, payload, signature — joined by dots. The payload carries **claims** (assertions about a subject), and the signature lets recipients verify the token was issued by the trusted party and unmodified.

## Notes
- Structure: `base64url(header).base64url(payload).base64url(signature)`.
- Signature algorithms: HMAC (HS256), RSA (RS256), ECDSA (ES256), EdDSA. Never trust `alg: none`.
- Standard claims: `iss` (issuer), `sub` (subject), `aud` (audience), `exp` (expiry), `nbf` (not before), `iat` (issued at), `jti` (JWT ID).
- Used as OIDC **ID Tokens** and as OAuth 2.0 access tokens (when not opaque).
- Verification: fetch the issuer's public key (JWKS endpoint), check signature, then validate `iss`/`aud`/`exp`/`nonce`.
- Pitfalls: tokens are *signed*, not *encrypted* — payload is readable; do not store secrets there. Revocation is hard because tokens are stateless — short expirations + refresh tokens compensate.

---

## 한국어

### 개요
JSON Web Token(JWT, RFC 7519)은 header, payload, signature 세 부분을 base64url로 인코딩해 점으로 이은 compact 토큰입니다. payload는 **claim**(주체에 대한 주장)을 담고, signature는 수신자가 토큰이 신뢰된 발급자에 의해 발급되었고 변조되지 않았음을 검증할 수 있게 합니다.

### 노트
- 구조: `base64url(header).base64url(payload).base64url(signature)`.
- 서명 알고리즘: HMAC (HS256), RSA (RS256), ECDSA (ES256), EdDSA. `alg: none`은 절대 신뢰하지 말 것.
- 표준 claim: `iss` (issuer), `sub` (subject), `aud` (audience), `exp` (expiry), `nbf` (not before), `iat` (issued at), `jti` (JWT ID).
- OIDC의 **ID Token**과 (opaque가 아닌) OAuth 2.0 access token으로 사용됩니다.
- 검증 절차: issuer의 공개키(JWKS 엔드포인트)를 가져와 signature 확인 후 `iss`/`aud`/`exp`/`nonce`를 검증합니다.
- 함정: 토큰은 *서명*된 것이지 *암호화*된 것이 아니어서 payload는 누구나 읽을 수 있으므로 비밀 정보를 담지 말 것. stateless라 무효화(revocation)가 어려우므로, 짧은 만료시간 + refresh token으로 보완합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
