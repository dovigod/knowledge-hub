---
id: 019e8d29-91df-749a-b371-a89eaf31b0a2
name: MX Record
aliases:
  - MX
  - MX 레코드
  - Mail Exchanger Record
updated_at: '2026-06-03T11:06:12.319Z'
summary: >-
  DNS record type that specifies the mail servers responsible for receiving
  email on behalf of a domain.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# MX Record

## Overview

An MX (Mail eXchanger) record is a DNS record type that designates which mail servers accept email for a domain. It is not a status value but a routing directive used by sending mail servers to find the recipient's inbound MTA.

## Notes

- Format: `example.com.  3600  IN  MX  10 mail1.example.com.`
- **Priority** — the leading integer (e.g. 10, 20). Lower values are tried first; equal values load-balance.
- The target **must** be a hostname resolvable via A/AAAA records. Pointing an MX at a CNAME or raw IP violates RFC 2181 / 5321.
- Common providers:
  - Google Workspace: `smtp.google.com` (priority 1)
  - Microsoft 365: `<domain>.mail.protection.outlook.com` (priority 0)
- Inspect with: `dig MX example.com +short`

---

## 한국어

### 개요

MX(Mail eXchanger) 레코드는 해당 도메인의 이메일을 수신할 메일 서버를 지정하는 DNS 레코드 타입이다. 상태값이 아니라, 발신 메일 서버가 수신 측 MTA를 찾기 위해 사용하는 라우팅 정보다.

### 노트

- 형식: `example.com.  3600  IN  MX  10 mail1.example.com.`
- **우선순위(Priority)** — 앞의 정수(예: 10, 20). 낮을수록 먼저 시도되고, 같은 값이면 부하 분산된다.
- 대상은 반드시 A/AAAA 레코드로 해석되는 호스트명이어야 한다. CNAME이나 IP 직접 지정은 RFC 2181 / 5321 위반.
- 대표 서비스 예시:
  - Google Workspace: `smtp.google.com` (우선순위 1)
  - Microsoft 365: `<도메인>.mail.protection.outlook.com` (우선순위 0)
- 확인 명령: `dig MX example.com +short`

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
