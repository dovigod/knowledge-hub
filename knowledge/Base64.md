---
id: 019e8ce6-0888-759d-8b30-b5df15aa3eb8
name: Base64
aliases:
  - Base64
  - b64
  - base-64
  - base64
updated_at: '2026-06-07T07:59:49.746Z'
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

> [!warning] Encoding ≠ encryption
> Base64 is fully reversible without a key. Anyone can decode it instantly — treating it as obfuscation is a classic mistake (see [[Basic Authentication]]).

## Notes
- **Not a security primitive.** Confidentiality must come from the transport layer (e.g. [[HTTPS]] / [[TLS]]), not from the encoding.
- Common uses: HTTP `Authorization` headers, [[JWT]] segments, data URIs, email attachments (MIME), embedding binary blobs in JSON/YAML.
- Output is ~33% larger than input.
- Variants: standard, URL-safe (`-` and `_` instead of `+` and `/`), no-padding.

## Examples
- `Authorization: Basic dXNlcjpwYXNz` → decodes to `user:pass` with a single `base64 -d`. The credentials are protected only while the [[HTTPS]] tunnel is up.

---

## 한국어

### 개요
Base64는 임의의 바이너리 데이터를 64자 ASCII 알파벳(A–Z, a–z, 0–9, +, /)과 `=` 패딩으로 인코딩하는 방식이다. HTTP 헤더, JSON, 이메일처럼 텍스트만 허용되는 채널로 바이너리를 안전하게 실어 나르기 위해 쓴다.

> [!warning] 인코딩은 암호화가 아니다
> Base64는 키 없이도 완전히 복호 가능하다. 디코드 한 번이면 원문이 그대로 드러나므로, 난독화 수단으로 착각하면 안 된다 ([[Basic Authentication]] 참고).

### 노트
- **보안 수단이 아니다.** 기밀성은 인코딩이 아니라 전송 계층([[HTTPS]] / [[TLS]])에서 확보해야 한다.
- 주요 용도: HTTP `Authorization` 헤더, [[JWT]] 세그먼트, data URI, 이메일 첨부(MIME), JSON/YAML 안에 바이너리 임베딩.
- 출력 크기는 입력 대비 약 33% 증가한다.
- 변형: 표준, URL-safe (`+`/`/` 대신 `-`/`_`), 패딩 없음.

### 예시
- `Authorization: Basic dXNlcjpwYXNz` → `base64 -d` 한 줄이면 `user:pass`로 복원된다. 자격증명이 보호되는 구간은 [[HTTPS]] 터널이 살아 있는 동안뿐이다.

## Sources

- [[raw/conversations/019e8ce5-afa5-74be-8636-3900cef4dbf2|019e8ce5-afa5-74be-8636-3900cef4dbf2]]
