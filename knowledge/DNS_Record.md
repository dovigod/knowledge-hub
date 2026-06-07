---
id: 019ea119-84b6-77e6-b181-43dad3e720dd
name: DNS Record
aliases:
  - DNS records
  - RR
  - dns 레코드
  - resource record
updated_at: '2026-06-07T08:01:04.695Z'
summary: >-
  An entry in a DNS zone that maps a name to a value (IP, hostname, mail server,
  etc.) and is identified by its record type such as A, AAAA, CNAME, MX, or TXT.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# DNS Record

## Overview

A **DNS record** (resource record, RR) is a single entry in a DNS zone that maps a name to a value. The record's **type** decides what the value means — an `A` record carries an IPv4 address, an `AAAA` carries IPv6, a `CNAME` aliases one name to another, an [[MX Record]] points to a mail server, a `TXT` carries arbitrary text used for verification, [[SPF]], [[DKIM]], and [[DMARC]].

> [!example] Anatomy of a record
> ```
> example.com.    3600    IN    A    93.184.216.34
>   ^name        ^TTL  ^class ^type ^rdata
> ```

## Notes

> [!note] Common record types
> - **A / AAAA** — name → IPv4 / IPv6
> - **CNAME** — name → another name (alias)
> - **MX** — mail routing (see [[MX Record]])
> - **TXT** — arbitrary strings; verification, [[SPF]], [[DKIM]], [[DMARC]]
> - **NS** — delegate a zone to authoritative nameservers
> - **PTR** — reverse lookup (IP → name)

> [!tip] Inspect with `dig`
> ```bash
> dig <type> <name> +short
> dig MX example.com +short
> ```

> [!warning] TTL matters
> TTL is how long resolvers may cache the answer. Lower it **before** you plan to change a record so the cutover propagates quickly.

---

## 한국어

### 개요

**DNS 레코드**(resource record, RR)는 DNS 존(zone) 안의 한 항목으로, 이름을 어떤 값에 매핑합니다. 레코드의 **타입**이 그 값의 의미를 결정합니다 — `A` 는 IPv4, `AAAA` 는 IPv6, `CNAME` 은 다른 이름으로의 별칭, [[MX Record]] 는 메일 서버, `TXT` 는 검증/[[SPF]]/[[DKIM]]/[[DMARC]] 같은 임의 문자열을 담습니다.

> [!example] 레코드 구조
> ```
> example.com.    3600    IN    A    93.184.216.34
>   ^이름        ^TTL  ^클래스 ^타입 ^값
> ```

### 노트

> [!note] 주요 레코드 타입
> - **A / AAAA** — 이름 → IPv4 / IPv6
> - **CNAME** — 이름 → 다른 이름 (별칭)
> - **MX** — 메일 라우팅 ([[MX Record]] 참고)
> - **TXT** — 임의 문자열; 도메인 검증, [[SPF]], [[DKIM]], [[DMARC]]
> - **NS** — 권한 있는 네임서버로 존 위임
> - **PTR** — 역방향 조회 (IP → 이름)

> [!tip] `dig` 로 확인
> ```bash
> dig <타입> <이름> +short
> dig MX example.com +short
> ```

> [!warning] TTL 주의
> TTL 은 리졸버가 응답을 캐시하는 시간입니다. 레코드를 변경하기 **전에** TTL 을 낮춰 두면 전환이 빠르게 전파됩니다.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
