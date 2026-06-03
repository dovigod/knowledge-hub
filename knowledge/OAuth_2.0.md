---
id: 019e8cf7-7c94-7497-a57f-9c1d30f127a4
name: OAuth 2.0
aliases:
  - OAuth
  - OAuth2
  - oauth-2.0
updated_at: '2026-06-03T10:11:30.068Z'
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
OAuth 2.0 is an authorization framework (RFC 6749) that delegates limited access to protected resources from a Resource Owner to a Client, mediated by an Authorization Server that issues Access Tokens. It explicitly does **not** define authentication — that gap is what OIDC fills.

## Notes
- Four roles: Resource Owner (user), Client (app), Authorization Server (issues tokens), Resource Server (hosts data).
- Grant types: Authorization Code (with PKCE), Client Credentials (machine-to-machine), Refresh Token, Device Code, Resource Owner Password (deprecated), Implicit (deprecated).
- Access tokens are typically opaque or JWT bearer tokens scoped to specific resources.
- PKCE (Proof Key for Code Exchange) is now mandatory for all clients, not just public ones (OAuth 2.1).
- Common pitfall: confusing OAuth 2.0 (authorization) with authentication — using an access token to identify a user is unsafe; use OIDC's ID Token instead.

---

## 한국어

### 개요
OAuth 2.0은 인가 프레임워크(RFC 6749)로, Authorization Server가 발급하는 Access Token을 매개로 Resource Owner의 보호된 리소스에 대한 제한된 접근 권한을 Client에게 위임합니다. 명시적으로 인증(authentication)은 정의하지 **않으며**, 이 공백을 OIDC가 채웁니다.

### 노트
- 4개 역할: Resource Owner (사용자), Client (앱), Authorization Server (토큰 발급), Resource Server (데이터 보관).
- Grant 타입: Authorization Code (PKCE 동반), Client Credentials (서버 간), Refresh Token, Device Code, Resource Owner Password (deprecated), Implicit (deprecated).
- Access token은 일반적으로 opaque 토큰 또는 특정 리소스에 scope이 지정된 JWT bearer 토큰입니다.
- PKCE(Proof Key for Code Exchange)는 OAuth 2.1에서 public 클라이언트뿐 아니라 모든 클라이언트에 필수입니다.
- 흔한 함정: OAuth 2.0(인가)과 인증을 혼동하는 것 — access token으로 사용자를 식별하는 것은 안전하지 않으며, OIDC의 ID Token을 사용해야 합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
