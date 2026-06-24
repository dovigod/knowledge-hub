---
id: 019e8d63-f27b-7527-8c47-610aa96ff4f2
name: DNS
aliases:
  - DNS
  - Domain Name System
  - dns
  - domain name system
updated_at: '2026-06-07T08:08:03.387Z'
summary: >-
  Hierarchical distributed naming system that maps human-readable domain names
  to IP addresses and other resource records.
sources:
  - 019e8d62-7f51-7099-87e6-cd054d1d0057
  - 019e8da4-1abf-712d-9a84-ecac84ea1a42
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# DNS

## Overview

**DNS (Domain Name System)** is the hierarchical, distributed database that resolves domain names (like `example.com`) to network resources — most commonly IP addresses, but also mail servers, text records, and service endpoints.

> [!note] Record types referenced here
> - **A / AAAA** — map a hostname to an IPv4 / IPv6 address.
> - **[[CNAME Record|CNAME]]** — alias one hostname to another domain name.
> - **[[MX Record|MX]]** — declares the mail server(s) for a domain.
> - **TXT** — free-form text; used for [[SPF]], [[DKIM]], [[DMARC]], domain verification.

## Notes

- Queries flow recursively from a resolver through root → TLD → authoritative servers.
- **TTL** controls how long a record may be cached; lower TTLs speed up propagation but increase query load.
- The `dig` and `nslookup` CLIs are the standard inspection tools.

## A Record vs CNAME

The **A record** and **[[CNAME Record|CNAME]]** are the two most commonly used DNS records, and they resolve names in fundamentally different ways.

**A record** — maps a hostname directly to an IPv4 address (**AAAA** does the same for IPv6).
- Example: `api.example.com → 1.2.3.4`
- The resolver returns the IP immediately in a single lookup.
- If the server's IP changes, the A record itself must be updated.

**CNAME record** — points a hostname at *another hostname* (an alias), not at an IP.
- Example: `www.example.com → app.hosting.com`
- The resolver follows the alias and performs a second lookup against the target to reach a final A / AAAA record.
- If the upstream service rotates its IPs, only the target hostname needs to stay correct — the CNAME does not change.

> [!example] Resolution flow
> - A record: `www.example.com → 1.2.3.4`
> - CNAME: `www.example.com → app.hosting.com → 1.2.3.4`

> [!warning] CNAME constraints
> - A CNAME chain must eventually terminate at an A or AAAA record — a CNAME alone cannot deliver an IP.
> - The same name cannot hold a CNAME *and* another record type (A, [[MX Record|MX]], TXT, …) simultaneously. This is why apex domains usually can't be CNAMEs.
> - Managed platforms like **Vercel** or **CloudFront** typically hand out a canonical hostname intended as a CNAME target.

> [!tip] Summary
> - **A** = domain → IP address
> - **CNAME** = domain → another domain name
> - At the end of any CNAME chain, an A or AAAA record provides the actual IP.

## MX Records

The **[[MX Record|MX]] (Mail eXchanger)** record is the DNS record type that declares which mail servers accept email for a domain. It is not a "status" — it is a routing pointer used by sending [[SMTP]] servers.

A zone may publish multiple MX records, each with a **priority** value:

```
example.com.    3600    IN    MX    10 mail1.example.com.
example.com.    3600    IN    MX    20 mail2.example.com.
```

- **Priority** — the leading integer (10, 20). **Lower values are tried first**; equal values are load-balanced. Above, `mail2` is only used as a fallback when `mail1` is unreachable.
- **Mail server hostname** — must resolve via an A / AAAA record. Pointing an MX at a [[CNAME Record|CNAME]] or directly at an IP literal violates the standard ([RFC 2181 §10.3](https://www.rfc-editor.org/rfc/rfc2181#section-10.3)).

> [!example] Common managed mail providers
> | Service | MX target | Priority |
> | --- | --- | --- |
> | Google Workspace | `smtp.google.com` | 1 |
> | Microsoft 365 | `<domain>.mail.protection.outlook.com` | 0 |

> [!tip] Inspecting MX records
> ```bash
> dig MX example.com +short
> ```
> Pair with [[SPF]] / [[DKIM]] / [[DMARC]] TXT lookups when diagnosing deliverability — MX alone tells you where mail goes, not whether it will be trusted on arrival.

---

## 한국어

### 개요

**DNS (Domain Name System)** 는 도메인 이름(`example.com` 같은)을 IP 주소나 메일 서버, 텍스트 레코드, 서비스 엔드포인트 등의 네트워크 리소스로 변환해 주는 계층적 분산 데이터베이스다.

> [!note] 본 문서에서 다루는 레코드 타입
> - **A / AAAA** — 호스트명을 IPv4 / IPv6 주소로 매핑.
> - **[[CNAME Record|CNAME]]** — 한 호스트명을 다른 호스트명의 별칭으로 지정.
> - **[[MX Record|MX]]** — 도메인의 메일 서버를 지정.
> - **TXT** — 자유 형식 텍스트로 [[SPF]], [[DKIM]], [[DMARC]], 도메인 소유 검증 등에 쓰임.

### 노트

- 질의는 리졸버 → 루트 → TLD → 권한 있는 서버 순으로 재귀적으로 전파된다.
- **TTL** 은 레코드의 캐시 보존 시간을 결정한다. TTL이 짧을수록 전파가 빠르지만 질의 부하가 늘어난다.
- `dig`, `nslookup` 이 표준 점검 도구다.

### A 레코드와 CNAME

A 레코드와 **[[CNAME Record|CNAME]]** 레코드는 DNS에서 가장 자주 쓰이는 두 가지 레코드이며, 이름을 해석하는 방식이 근본적으로 다르다.

**A 레코드** — 호스트명을 IPv4 주소에 직접 연결한다 (IPv6의 경우 **AAAA**).
- 예: `api.example.com → 1.2.3.4`
- DNS 조회 시 한 번에 IP 주소를 반환한다.
- 서버 IP가 바뀌면 A 레코드를 직접 수정해야 한다.

**CNAME 레코드** — 호스트명을 IP가 아닌 *다른 호스트명*(별칭, alias)에 연결한다.
- 예: `www.example.com → app.hosting.com`
- 리졸버가 별칭을 따라가 대상 도메인을 한 번 더 조회하여 최종 A / AAAA 레코드로 도달한다.
- 대상 서비스가 IP를 바꿔도 CNAME 대상 도메인만 유지되면 되고, CNAME 자체는 건드릴 필요가 없다.

> [!example] 조회 흐름 비교
> - A 레코드: `www.example.com → 1.2.3.4`
> - CNAME: `www.example.com → app.hosting.com → 1.2.3.4`

> [!warning] CNAME 제약
> - CNAME 체인은 결국 A 또는 AAAA 레코드에서 끝나야 한다 — CNAME 단독으로는 IP를 제공할 수 없다.
> - 같은 이름에 CNAME과 다른 레코드(A, [[MX Record|MX]], TXT 등)를 동시에 둘 수 없다. 그래서 apex 도메인은 보통 CNAME으로 둘 수 없다.
> - **Vercel**, **CloudFront** 같은 매니지드 서비스는 보통 CNAME 대상으로 쓰라고 canonical hostname을 제공한다.

> [!tip] 정리
> - **A 레코드** = 도메인 → IP 주소
> - **CNAME** = 도메인 → 다른 도메인 이름
> - CNAME 체인의 끝에서 A 또는 AAAA 레코드가 실제 IP를 제공한다.

### MX 레코드

**[[MX Record|MX]] (Mail eXchanger)** 레코드는 "상태값"이 아니라, 해당 도메인의 메일을 수신할 메일 서버를 지정하는 DNS 레코드 타입이다. 보내는 [[SMTP]] 서버가 수신지를 찾을 때 참조하는 라우팅 포인터다.

한 존(zone)에는 여러 개의 MX 레코드를 둘 수 있고, 각각 **우선순위(priority)** 값을 가진다.

```
example.com.    3600    IN    MX    10 mail1.example.com.
example.com.    3600    IN    MX    20 mail2.example.com.
```

- **우선순위** — 앞의 정수(10, 20). **값이 낮을수록 먼저 시도**되고, 동일하면 부하 분산된다. 위 예시에서는 `mail1` 이 불가할 때만 `mail2` 로 폴백된다.
- **메일 서버 호스트명** — 반드시 A / AAAA 레코드로 해석되는 호스트명이어야 한다. MX 대상을 [[CNAME Record|CNAME]] 으로 두거나 IP를 직접 박는 것은 표준 위반이다 ([RFC 2181 §10.3](https://www.rfc-editor.org/rfc/rfc2181#section-10.3)).

> [!example] 대표적인 매니지드 메일 서비스
> | 서비스 | MX 대상 | 우선순위 |
> | --- | --- | --- |
> | Google Workspace | `smtp.google.com` | 1 |
> | Microsoft 365 | `<domain>.mail.protection.outlook.com` | 0 |

> [!tip] MX 레코드 확인
> ```bash
> dig MX example.com +short
> ```
> 메일 도달성(deliverability)을 점검할 때는 [[SPF]] / [[DKIM]] / [[DMARC]] TXT 조회와 함께 보는 것이 좋다 — MX는 "어디로 가는지"만 알려주지, "수신측이 신뢰할지"는 말해 주지 않는다.

## Sources

- [[raw/conversations/019e8d62-7f51-7099-87e6-cd054d1d0057|019e8d62-7f51-7099-87e6-cd054d1d0057]]
- [[raw/conversations/019e8da4-1abf-712d-9a84-ecac84ea1a42|019e8da4-1abf-712d-9a84-ecac84ea1a42]]
