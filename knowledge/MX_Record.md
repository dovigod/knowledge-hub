---
id: 019e8d29-91df-749a-b371-a89eaf31b0a2
name: MX Record
aliases:
  - MX
  - MX Record
  - MX 레코드
  - Mail Exchanger Record
  - Mail eXchanger
  - mail exchanger
  - mail exchanger record
  - mx
  - mx record
  - mx 레코드
updated_at: '2026-06-07T08:01:00.919Z'
summary: >-
  DNS record type that specifies the mail servers responsible for receiving
  email on behalf of a domain.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
  - 019e8d62-7f51-7099-87e6-cd054d1d0057
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# MX Record

## Overview

An MX (Mail eXchanger) record is a DNS record type that designates which mail servers accept email for a domain. It is not a status value — unlike a [[JobState]] field — but a routing directive used by sending mail servers to find the recipient's inbound MTA.

> [!note] Lower priority wins
> The leading integer is preference, not weight. Lower numbers are tried first; equal values load-balance across hosts.

> [!tip] MX is routing, not deliverability
> A valid MX gets mail to the right server, but whether it lands in the inbox depends on the sending host's [[IP Reputation]], SPF/DKIM/DMARC alignment, and content signals.

## Notes

- Format: `example.com.  3600  IN  MX  10 mail1.example.com.`
- **Priority** — the leading integer (e.g. 10, 20). Lower values are tried first; equal values load-balance.
- The target **must** be a hostname resolvable via A/AAAA records. Pointing an MX at a CNAME or raw IP violates RFC 2181 / 5321.
- Inspect with: `dig MX example.com +short`
- MX only declares *where* to deliver; receiver-side acceptance is governed by [[IP Reputation]] and authentication records (SPF/DKIM/DMARC).

> [!warning] No CNAME, no IP
> MX targets that resolve through a CNAME or point directly at an IP are non-compliant and will be rejected or silently downgraded by strict MTAs.

## Examples

```
example.com.    3600    IN    MX    10 mail1.example.com.
example.com.    3600    IN    MX    20 mail2.example.com.
```

Failover: mail2 is only used when mail1 is unreachable.

Common providers:

| Service | MX target | Priority |
|---|---|---|
| Google Workspace | `smtp.google.com` | 1 |
| Microsoft 365 | `<domain>.mail.protection.outlook.com` | 0 |

## See also

- [[IP Reputation]] — what determines whether mail leaving these MX-pointed servers actually reaches inboxes.
- [[JobState]] — the in-repo `"pending" | "running" | "done" | "failed"` enum that "mx 상태값" was initially confused with.

---

## 한국어

### 개요

MX(Mail eXchanger) 레코드는 해당 도메인의 이메일을 수신할 메일 서버를 지정하는 DNS 레코드 타입이다. [[JobState]] 같은 상태값이 아니라, 발신 메일 서버가 수신 측 MTA를 찾기 위해 사용하는 라우팅 정보다.

> [!note] 우선순위는 낮을수록 먼저
> 앞의 정수는 가중치가 아니라 선호도다. 낮은 값이 먼저 시도되고, 같은 값이면 호스트 간 부하 분산된다.

> [!tip] MX는 라우팅, 도달률(deliverability)이 아니다
> MX가 올바르면 메일은 "맞는 서버"로 가지만, 실제 인박스 도달 여부는 발신 호스트의 [[IP Reputation]], SPF/DKIM/DMARC 정합성, 본문 신호 등에 달려 있다.

### 노트

- 형식: `example.com.  3600  IN  MX  10 mail1.example.com.`
- **우선순위(Priority)** — 앞의 정수(예: 10, 20). 낮을수록 먼저 시도되고, 같은 값이면 부하 분산된다.
- 대상은 반드시 A/AAAA 레코드로 해석되는 호스트명이어야 한다. CNAME이나 IP 직접 지정은 RFC 2181 / 5321 위반.
- 확인 명령: `dig MX example.com +short`
- MX는 "어디로 보낼지"만 정의한다. 수신 측이 받아주느냐는 [[IP Reputation]]과 인증 레코드(SPF/DKIM/DMARC)가 결정한다.

> [!warning] CNAME·IP 직접 지정 금지
> CNAME을 거치거나 IP를 직접 지정한 MX 대상은 표준 위반이며, 엄격한 MTA는 거부하거나 조용히 다운그레이드한다.

### 예시

```
example.com.    3600    IN    MX    10 mail1.example.com.
example.com.    3600    IN    MX    20 mail2.example.com.
```

페일오버: mail1이 닿지 않을 때만 mail2가 사용된다.

대표 서비스 예시:

| 서비스 | MX 값 | 우선순위 |
|---|---|---|
| Google Workspace | `smtp.google.com` | 1 |
| Microsoft 365 | `<도메인>.mail.protection.outlook.com` | 0 |

### 관련 항목

- [[IP Reputation]] — 위 MX가 가리키는 서버에서 보낸 메일이 실제 인박스까지 도달하는지를 좌우한다.
- [[JobState]] — "mx 상태값"으로 처음 혼동했던 저장소 내부의 `"pending" | "running" | "done" | "failed"` 잡 상태 열거형.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
- [[raw/conversations/019e8d62-7f51-7099-87e6-cd054d1d0057|019e8d62-7f51-7099-87e6-cd054d1d0057]]
