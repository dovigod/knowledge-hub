---
id: 019e8ce6-1c0a-75ee-acfc-9c5397c5635a
name: HTTPS
aliases:
  - HTTP over TLS
  - SSL
  - TLS
updated_at: '2026-06-03T09:52:31.242Z'
summary: 'HTTP over TLS — the standard for encrypted, authenticated web traffic.'
sources:
  - 019e8ce5-afa5-74be-8636-3900cef4dbf2
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# HTTPS

## Overview
HTTPS wraps HTTP in a TLS-encrypted channel, providing confidentiality, integrity, and server authentication (and optionally client authentication via mTLS).

## Notes
- **Required for any credential transmission**, including [[Basic Authentication]], cookies, and bearer tokens.
- Protects the entire HTTP message (headers + body) from network observers — but not from the endpoints themselves or from anything that terminates TLS (load balancers, CDNs).
- Modern baseline: TLS 1.2+ with forward-secret cipher suites; TLS 1.3 preferred.

## Sources

- [[raw/conversations/019e8ce5-afa5-74be-8636-3900cef4dbf2|019e8ce5-afa5-74be-8636-3900cef4dbf2]]
