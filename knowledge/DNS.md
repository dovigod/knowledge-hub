---
id: 019e8d63-f27b-7527-8c47-610aa96ff4f2
name: DNS
aliases:
  - Domain Name System
  - dns
  - domain name system
updated_at: '2026-06-03T12:09:58.140Z'
summary: >-
  Hierarchical distributed naming system that maps human-readable domain names
  to IP addresses and other resource records.
sources:
  - 019e8d62-7f51-7099-87e6-cd054d1d0057
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# DNS

## Overview

**DNS (Domain Name System)** is the hierarchical, distributed database that resolves domain names (like `example.com`) to network resources — most commonly IP addresses, but also mail servers, text records, and service endpoints.

> [!note] Record types referenced here
> - **A / AAAA** — map a hostname to an IPv4 / IPv6 address.
> - **[[MX Record|MX]]** — declares the mail server(s) for a domain.
> - **[[CNAME Record|CNAME]]** — alias one hostname to another.
> - **TXT** — free-form text; used for [[SPF]], [[DKIM]], [[DMARC]], domain verification.

## Notes

- Queries flow recursively from a resolver through root → TLD → authoritative servers.
- **TTL** controls how long a record may be cached; lower TTLs speed up propagation but increase query load.
- The `dig` and `nslookup` CLIs are the standard inspection tools.

---

## 한국어

### 개요

**DNS (Domain Name System)** 는 도메인 이름(`example.com` 같은)을 IP 주소나 메일 서버, 텍스트 레코드, 서비스 엔드포인트 등의 네트워크 리소스로 변환해 주는 계층적 분산 데이터베이스다.

> [!note] 본 문서에서 다루는 레코드 타입
> - **A / AAAA** — 호스트명을 IPv4 / IPv6 주소로 매핑.
> - **[[MX Record|MX]]** — 도메인의 메일 서버를 지정.
> - **[[CNAME Record|CNAME]]** — 한 호스트명을 다른 호스트명의 별칭으로 지정.
> - **TXT** — 자유 형식 텍스트로 [[SPF]], [[DKIM]], [[DMARC]], 도메인 소유 검증 등에 쓰임.

### 노트

- 질의는 리졸버 → 루트 → TLD → 권한 있는 서버 순으로 재귀적으로 전파된다.
- **TTL** 은 레코드의 캐시 보존 시간을 결정한다. TTL이 짧을수록 전파가 빠르지만 질의 부하가 늘어난다.
- `dig`, `nslookup` 이 표준 점검 도구다.

## Sources

- [[raw/conversations/019e8d62-7f51-7099-87e6-cd054d1d0057|019e8d62-7f51-7099-87e6-cd054d1d0057]]
