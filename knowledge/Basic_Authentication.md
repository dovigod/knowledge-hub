---
id: 019e8ce6-0885-74e3-900b-e29c77d9c947
name: Basic Authentication
aliases:
  - Basic Auth
  - HTTP Basic Auth
  - basic auth
updated_at: '2026-06-03T09:52:26.245Z'
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

## Notes
- **Base64 is encoding, not encryption.** Anyone who intercepts the header can decode it trivially (`base64 -d`). It provides zero confidentiality on its own.
- **HTTPS is mandatory.** Basic Auth is only safe when the transport is TLS-encrypted — TLS protects the header in transit, not Basic Auth itself.
- **No session/logout semantics.** Credentials are resent on every request, increasing exposure surface.
- **When it's still acceptable:** internal service-to-service calls over TLS, simple admin endpoints, MCP/API endpoints behind a reverse proxy, or as a transitional auth layer.
- **When to avoid:** public-facing user login (use OAuth/OIDC/session cookies), any non-TLS context, anywhere credentials might be logged by proxies/CDNs.
- Common alternatives: Bearer tokens (JWT/OAuth), API keys with HMAC signing, mTLS, session cookies.

## Sources

- [[raw/conversations/019e8ce5-afa5-74be-8636-3900cef4dbf2|019e8ce5-afa5-74be-8636-3900cef4dbf2]]
