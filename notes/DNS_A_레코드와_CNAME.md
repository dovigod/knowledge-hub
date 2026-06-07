---
id: 019e8dac-857f-77fa-90b9-c7147e4917cf
title: DNS MX 레코드
topics:
  - dns
  - a-record
  - cname
  - 네트워크
  - mx
  - email
  - 메일 서버
sources:
  - 019e8da4-1abf-712d-9a84-ecac84ea1a42
  - 019e8d28-0dc9-755c-834d-aad29d26a380
created_at: '2026-06-03T13:29:14.367Z'
updated_at: '2026-06-07T08:01:30.768Z'
---
## A Record

- Maps a domain name directly to an IPv4 address.
- Example:
  api.example.com → 1.2.3.4
- DNS lookup returns the IP address immediately.
- If the server IP changes, the A record must be updated.

## CNAME Record

- Maps a domain name to another domain name as an alias.
- Example:
  www.example.com → app.hosting.com
- DNS lookup performs an additional lookup on the target domain to find the final IP.
- Even if the target service changes its IP, you only need to manage the CNAME target domain.

## Lookup Flow Comparison

- A record:
  www.example.com → 1.2.3.4
- CNAME:
  www.example.com → app.hosting.com → 1.2.3.4

## Important Points

- A [[CNAME]] points to another domain name, and at the end of the lookup chain there must be an [[A record]] or [[AAAA record]].
- You cannot have a CNAME and other records (A, MX, etc.) on the same name at the same time.
- Services like [[Vercel]] and [[CloudFront]] usually provide a canonical hostname as the CNAME target.

## Summary

- A record = domain → IP address
- CNAME = domain → another domain name
- Ultimately, at the end of the CNAME chain, an A or AAAA record provides the actual IP.

## MX Record

**[[MX]] (Mail eXchanger) record** is a DNS record type — not a status value — that **specifies the mail server that will receive email for the domain**.

Structure:

```
example.com.    3600    IN    MX    10 mail1.example.com.
example.com.    3600    IN    MX    20 mail2.example.com.
```

- **Priority** — the leading number (10, 20). **Lower values are tried first**, and equal values are load-balanced. In the example above, mail2 is only used when mail1 is down.
- **Mail server hostname** — must be a hostname that resolves via an A/AAAA record. Pointing to a CNAME or to an IP address directly is a standard violation.

Common usage examples:

| Service | MX value |
|---|---|
| Google Workspace | `smtp.google.com` (priority 1) |
| Microsoft 365 | `<domain>.mail.protection.outlook.com` (0) |

How to check:

```bash
dig MX example.com +short
```

---

## 한국어

### A 레코드

- 도메인 이름을 IPv4 주소에 직접 연결합니다.
- 예:
  api.example.com → 1.2.3.4
- [[DNS]] 조회 시 바로 IP 주소를 반환합니다.
- 서버 IP가 바뀌면 A 레코드를 수정해야 합니다.

### CNAME 레코드

- 도메인 이름을 다른 도메인 이름의 별칭(alias)으로 연결합니다.
- 예:
  www.example.com → app.hosting.com
- DNS 조회 시 대상 도메인을 한 번 더 조회하여 최종 IP를 찾습니다.
- 대상 서비스가 IP를 변경해도 CNAME 대상 도메인만 관리하면 됩니다.

### 조회 흐름 비교

- A 레코드:
  www.example.com → 1.2.3.4
- CNAME:
  www.example.com → app.hosting.com → 1.2.3.4

### 중요한 점

- [[CNAME]]은 다른 도메인 이름을 가리키며, 결국 조회 과정의 끝에는 [[A 레코드]] 또는 [[AAAA 레코드]]가 존재해야 합니다.
- 같은 이름에는 CNAME과 다른 레코드(A, MX 등)를 동시에 둘 수 없습니다.
- [[Vercel]], [[CloudFront]] 같은 서비스는 보통 CNAME 대상으로 제공하는 canonical hostname을 제공합니다.

### 정리

- A 레코드 = 도메인 → IP 주소
- CNAME = 도메인 → 다른 도메인 이름
- 최종적으로는 CNAME 체인의 끝에서 A 또는 AAAA 레코드가 실제 IP를 제공합니다.

### MX 레코드

**[[MX]] (Mail eXchanger) 레코드**는 상태값이라기보다는 DNS 레코드 타입 중 하나로, **해당 도메인의 이메일을 수신할 메일 서버를 지정**하는 레코드입니다.

구조는 이렇습니다:

```
example.com.    3600    IN    MX    10 mail1.example.com.
example.com.    3600    IN    MX    20 mail2.example.com.
```

- **우선순위 (Priority)** — 앞의 숫자 (10, 20). **낮을수록 먼저 시도**되고, 같은 값이면 부하 분산됩니다. 위 예시에선 mail1이 죽었을 때만 mail2로 갑니다.
- **메일 서버 호스트명** — 반드시 A/AAAA 레코드로 해석되는 호스트명이어야 하고, CNAME이나 IP 직접 지정은 표준 위반입니다.

흔한 사용 예:

| 서비스 | MX 값 |
|---|---|
| Google Workspace | `smtp.google.com` (우선순위 1) |
| Microsoft 365 | `<도메인>.mail.protection.outlook.com` (0) |

확인 방법:

```bash
dig MX example.com +short
```
