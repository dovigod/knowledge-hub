---
id: 019e8ce6-0885-74e3-900b-e29c77d9c947
name: Basic Authentication
aliases:
  - Basic Auth
  - Basic Authentication
  - HTTP Basic Auth
  - basic auth
  - 베이직 인증
updated_at: '2026-06-07T07:59:32.798Z'
summary: >-
  An HTTP authentication scheme that transmits credentials as a Base64-encoded
  username:password string in the Authorization header.
sources:
  - 019e8ce5-afa5-74be-8636-3900cef4dbf2
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Basic Authentication

## Overview
Basic Authentication is a simple HTTP authentication scheme defined in RFC 7617. The client sends credentials in the `Authorization: Basic <base64(username:password)>` header.

> [!warning] Base64 ≠ encryption
> Anyone who intercepts the header can decode it trivially with `base64 -d`. Basic Auth provides **zero confidentiality** on its own — its safety depends entirely on the transport layer.

## Notes
- **Base64 is encoding, not encryption.** The header is fully reversible — `base64 -d` reveals `username:password` in plaintext. This is the most common misconception about Basic Auth.
- **HTTPS is mandatory.** Basic Auth is only safe when the transport is TLS-encrypted — TLS protects the header in transit, not Basic Auth itself. Without [[HTTPS]]/[[TLS]], credentials are effectively sent in cleartext.
- **No session/logout semantics.** Credentials are resent on every request, increasing exposure surface.
- **When it's still acceptable:** internal service-to-service calls over TLS, simple admin endpoints, [[MCP]]/API endpoints behind a reverse proxy, or as a transitional auth layer.
- **When to avoid:** public-facing user login (use [[OAuth]]/[[OIDC]]/session cookies), any non-TLS context, anywhere credentials might be logged by proxies/CDNs.
- Common alternatives: [[Bearer Token]] ([[JWT]]/[[OAuth]]), API keys with HMAC signing, [[mTLS]], session cookies.

> [!tip] Decision rule
> If the channel is TLS-terminated end-to-end and the consumer is a trusted service (not a browser user), Basic Auth is fine. Otherwise, reach for a bearer token or session-based scheme.

---

## 한국어

### 개요
Basic Authentication은 RFC 7617에 정의된 단순한 HTTP 인증 방식이다. 클라이언트는 `Authorization: Basic <base64(username:password)>` 헤더로 자격증명을 전송한다.

> [!warning] Base64는 암호화가 아니다
> 헤더를 가로챈 사람은 `base64 -d` 한 줄로 즉시 복호화할 수 있다. Basic Auth 자체는 **기밀성을 전혀 제공하지 않으며**, 안전성은 전적으로 전송 계층에 의존한다.

### 노트
- **Base64는 인코딩이지 암호화가 아니다.** 헤더는 완전히 가역적이라 `base64 -d` 한 번으로 `username:password`가 평문으로 드러난다. Basic Auth에 대한 가장 흔한 오해다.
- **HTTPS는 필수다.** Basic Auth는 전송 구간이 TLS로 암호화될 때만 안전하다 — 헤더를 보호하는 것은 TLS이지 Basic Auth가 아니다. [[HTTPS]]/[[TLS]] 없이는 자격증명이 사실상 평문으로 전송된다.
- **세션/로그아웃 의미가 없다.** 매 요청마다 자격증명이 재전송되어 노출 표면이 커진다.
- **여전히 허용되는 경우:** TLS 위의 내부 서비스 간 호출, 단순한 admin 엔드포인트, 리버스 프록시 뒤의 [[MCP]]/API 엔드포인트, 또는 과도기적 인증 레이어.
- **피해야 할 경우:** 공개 사용자 로그인 (대신 [[OAuth]]/[[OIDC]]/세션 쿠키 사용), TLS가 없는 모든 환경, 프록시/CDN이 자격증명을 로깅할 가능성이 있는 곳.
- 대안: [[Bearer Token]] ([[JWT]]/[[OAuth]]), HMAC 서명을 동반한 API 키, [[mTLS]], 세션 쿠키.

> [!tip] 판단 기준
> 채널이 종단 간 TLS로 종료되고 소비자가 (브라우저 사용자가 아닌) 신뢰된 서비스라면 Basic Auth는 괜찮다. 그 외에는 bearer token이나 세션 기반 방식을 선택하라.

## Sources

- [[raw/conversations/019e8ce5-afa5-74be-8636-3900cef4dbf2|019e8ce5-afa5-74be-8636-3900cef4dbf2]]
