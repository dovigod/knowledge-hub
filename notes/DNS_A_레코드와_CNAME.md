---
id: 019e8dac-857f-77fa-90b9-c7147e4917cf
title: DNS A 레코드와 CNAME
topics:
  - dns
  - a-record
  - cname
  - 네트워크
sources:
  - 019e8da4-1abf-712d-9a84-ecac84ea1a42
created_at: '2026-06-03T13:29:14.367Z'
updated_at: '2026-06-03T13:29:14.367Z'
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
