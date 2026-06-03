---
id: 019e8da5-1b18-7632-b15b-f2181ff0e0d2
name: CNAME Record
aliases:
  - CNAME
  - CNAME 레코드
  - Canonical Name Record
  - cname
updated_at: '2026-06-03T13:21:08.376Z'
summary: >-
  A DNS record that aliases one domain name to another canonical domain name,
  deferring final IP resolution.
sources:
  - 019e8da4-1abf-712d-9a84-ecac84ea1a42
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# CNAME Record

## Overview

A **CNAME Record** (Canonical Name) aliases one domain name to another domain name, so DNS resolution follows the alias chain until it lands on an [[A Record]] or [[AAAA Record]].

> [!note] Core mapping
> `www.example.com → app.hosting.com → 1.2.3.4`

## Notes

- Points to another domain name, not an IP — resolution requires an extra lookup hop.
- Useful when the target service (e.g. [[Vercel]], [[CloudFront]]) manages its own IPs; you only maintain the alias.
- Managed hosting providers typically publish a canonical hostname for you to set as the CNAME target.
- The chain must terminate at an [[A Record]] (IPv4) or [[AAAA Record]] (IPv6) to actually resolve to an IP.

> [!warning] Coexistence rule
> The same name cannot hold a CNAME alongside any other record type (A, MX, etc.) — RFC restriction.

> [!tip] Operational benefit
> If the upstream service changes its IPs, you don't touch your DNS — the canonical target absorbs the change.

> [!example] Typical setup
> Point `www.example.com` via CNAME to a platform's canonical hostname (e.g. `cname.vercel-dns.com`) instead of hardcoding their IP.

---

## 한국어

### 개요

**CNAME 레코드**(Canonical Name)는 하나의 도메인 이름을 다른 도메인 이름의 별칭으로 연결하므로, DNS 해석은 별칭 체인을 따라가다 최종적으로 [[A Record]] 또는 [[AAAA Record]]에 도달합니다.

> [!note] 핵심 매핑
> `www.example.com → app.hosting.com → 1.2.3.4`

### 노트

- IP가 아니라 다른 도메인 이름을 가리키므로 해석 시 추가 조회 단계가 필요합니다.
- 대상 서비스([[Vercel]], [[CloudFront]] 등)가 자체적으로 IP를 관리할 때 유용하며, 사용자는 별칭만 관리하면 됩니다.
- 매니지드 호스팅 제공자는 보통 CNAME 대상으로 설정할 canonical hostname을 제공합니다.
- 체인은 결국 [[A Record]](IPv4) 또는 [[AAAA Record]](IPv6)에서 끝나야 실제 IP로 해석됩니다.

> [!warning] 공존 규칙
> 동일한 이름에 CNAME과 다른 레코드 타입(A, MX 등)을 함께 둘 수 없습니다 — RFC 제한.

> [!tip] 운영상의 이점
> 상위 서비스가 IP를 변경해도 사용자의 DNS는 손대지 않아도 되며, canonical 대상이 변경을 흡수합니다.

> [!example] 일반적인 설정
> 플랫폼의 IP를 하드코딩하는 대신, `www.example.com`을 CNAME으로 해당 플랫폼의 canonical hostname(예: `cname.vercel-dns.com`)에 연결합니다.

## Sources

- [[raw/conversations/019e8da4-1abf-712d-9a84-ecac84ea1a42|019e8da4-1abf-712d-9a84-ecac84ea1a42]]
