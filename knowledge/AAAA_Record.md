---
id: 019e8da5-9abd-7318-a1c4-7715f5d992c8
name: AAAA Record
aliases:
  - AAAA
  - AAAA 레코드
  - IPv6 record
  - quad-A record
updated_at: '2026-06-03T13:21:41.053Z'
summary: A DNS record type that maps a domain name directly to an IPv6 address.
sources:
  - 019e8da4-1abf-712d-9a84-ecac84ea1a42
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# AAAA Record

## Overview

An **AAAA Record** is the IPv6 counterpart of the [[A Record]], mapping a domain name directly to a 128-bit IPv6 address.

> [!note] Core mapping
> `api.example.com → 2001:db8::1`

## Notes

- Identical role to the [[A Record]], except the value is an IPv6 address rather than IPv4.
- A name can carry both A and AAAA records; resolvers pick based on client capability and preference.
- [[CNAME Record]] chains terminate at either an [[A Record]] or AAAA Record to yield a final IP.

> [!tip] Deployment
> Publish both A and AAAA records when your origin supports dual-stack, so IPv6-capable clients can connect without translation.

---

## 한국어

### 개요

**AAAA 레코드**는 [[A Record]]의 IPv6 대응 레코드로, 도메인 이름을 128비트 IPv6 주소에 직접 매핑합니다.

> [!note] 핵심 매핑
> `api.example.com → 2001:db8::1`

### 노트

- 역할은 [[A Record]]와 동일하지만 값이 IPv4가 아닌 IPv6 주소입니다.
- 한 이름에 A와 AAAA 레코드를 동시에 둘 수 있으며, 리졸버는 클라이언트의 능력과 선호에 따라 선택합니다.
- [[CNAME Record]] 체인은 최종 IP를 얻기 위해 [[A Record]] 또는 AAAA 레코드에서 종료됩니다.

> [!tip] 배포
> 오리진이 듀얼 스택을 지원할 때 A와 AAAA를 함께 게시하면, IPv6 가능 클라이언트가 변환 없이 접속할 수 있습니다.

## Sources

- [[raw/conversations/019e8da4-1abf-712d-9a84-ecac84ea1a42|019e8da4-1abf-712d-9a84-ecac84ea1a42]]
