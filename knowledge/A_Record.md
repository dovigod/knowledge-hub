---
id: 019e8da5-0ac4-7302-9dd1-9de7c5a31e0a
name: A Record
aliases:
  - A record
  - A 레코드
  - DNS A record
  - a-record
updated_at: '2026-06-03T13:21:04.196Z'
summary: A DNS record type that maps a domain name directly to an IPv4 address.
sources:
  - 019e8da4-1abf-712d-9a84-ecac84ea1a42
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# A Record

## Overview

An **A Record** is a DNS record that maps a domain name directly to an IPv4 address, returning the IP in a single lookup.

> [!note] Core mapping
> `api.example.com → 1.2.3.4`

## Notes

- Resolves a hostname to a single IPv4 address in one lookup step.
- If the server's IP changes, the A record itself must be updated.
- For IPv6, the equivalent is the [[AAAA Record]].
- Compare with [[CNAME Record]], which points to another domain name rather than an IP.
- Every [[CNAME Record]] chain must ultimately terminate at an A or [[AAAA Record]] to resolve.

> [!tip] When to use
> Use an A record when you control a stable IP address and want the fastest, most direct resolution.

> [!warning] Coexistence rule
> The same name cannot hold both an A record and a [[CNAME Record]] simultaneously.

---

## 한국어

### 개요

**A 레코드**는 도메인 이름을 IPv4 주소에 직접 매핑하는 DNS 레코드로, 한 번의 조회로 IP를 반환합니다.

> [!note] 핵심 매핑
> `api.example.com → 1.2.3.4`

### 노트

- 호스트명을 한 번의 조회 단계로 단일 IPv4 주소로 해석합니다.
- 서버의 IP가 바뀌면 A 레코드 자체를 수정해야 합니다.
- IPv6의 경우 대응되는 레코드는 [[AAAA Record]]입니다.
- IP가 아니라 다른 도메인 이름을 가리키는 [[CNAME Record]]와 비교됩니다.
- 모든 [[CNAME Record]] 체인은 최종적으로 A 또는 [[AAAA Record]]에서 끝나야 해석이 완료됩니다.

> [!tip] 사용 시점
> 안정적인 IP 주소를 직접 관리하고 가장 빠르고 직접적인 해석이 필요할 때 A 레코드를 사용합니다.

> [!warning] 공존 규칙
> 동일한 이름에 A 레코드와 [[CNAME Record]]를 동시에 둘 수 없습니다.

## Sources

- [[raw/conversations/019e8da4-1abf-712d-9a84-ecac84ea1a42|019e8da4-1abf-712d-9a84-ecac84ea1a42]]
