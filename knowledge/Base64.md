---
id: 019e8ce6-0888-759d-8b30-b5df15aa3eb8
name: Base64
aliases:
  - b64
  - base-64
updated_at: '2026-06-03T09:52:26.248Z'
summary: >-
  A binary-to-text encoding scheme that represents binary data using 64 ASCII
  characters; it is encoding, not encryption.
sources:
  - 019e8ce5-afa5-74be-8636-3900cef4dbf2
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Base64

## Overview
Base64 encodes arbitrary binary data into a 64-character ASCII alphabet (A–Z, a–z, 0–9, +, /) plus `=` padding. It is used to safely transport binary data through text-only channels (HTTP headers, JSON, email).

## Notes
- **Not a security primitive.** Base64 is fully reversible without a key. Treating it as obfuscation is a classic mistake (see [[Basic Authentication]]).
- Common uses: HTTP `Authorization` headers, JWT segments, data URIs, email attachments (MIME), embedding binary blobs in JSON/YAML.
- Output is ~33% larger than input.
- Variants: standard, URL-safe (`-` and `_` instead of `+` and `/`), no-padding.

## Sources

- [[raw/conversations/019e8ce5-afa5-74be-8636-3900cef4dbf2|019e8ce5-afa5-74be-8636-3900cef4dbf2]]
