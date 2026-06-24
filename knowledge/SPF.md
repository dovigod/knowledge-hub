---
id: 019e8d29-c2f6-73a3-9228-3ae96fdea9b3
name: SPF
aliases:
  - Sender Policy Framework
updated_at: '2026-06-03T11:06:24.886Z'
summary: >-
  Email authentication standard that lets a domain owner declare, via DNS, which
  IP addresses are authorized to send mail for that domain.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# SPF

## Overview

SPF (Sender Policy Framework) is an email authentication mechanism that lets a domain owner publish, via a DNS TXT record, the list of IP addresses or hosts authorized to send mail on behalf of the domain. Receiving servers check the sending IP against this list to detect spoofing.

## Notes

- Published as a DNS TXT record on the envelope-from (`MAIL FROM`) domain.
- Combined with DKIM and DMARC, it forms the baseline authentication stack now required by Gmail and Yahoo bulk-sender rules.
- An SPF pass does not by itself protect the visible `From:` header — DMARC alignment is what ties SPF/DKIM results to the user-visible domain.
- Must be kept in sync with the set of ESPs/relays that actually send for the domain; stale SPF records are a common deliverability cause.

---

## 한국어

### 개요

SPF(Sender Policy Framework)는 도메인 소유자가 해당 도메인을 대신해 메일을 발송할 수 있는 IP 주소 또는 호스트 목록을 DNS TXT 레코드로 공표하는 이메일 인증 메커니즘이다. 수신 서버는 발신 IP가 이 목록에 포함되는지 확인해 스푸핑 여부를 판단한다.

### 노트

- envelope-from(`MAIL FROM`) 도메인의 DNS TXT 레코드로 게시한다.
- DKIM, DMARC와 결합되어 Gmail·Yahoo 대량발송자 정책이 요구하는 기본 인증 스택을 구성한다.
- SPF pass만으로는 사용자에게 보이는 `From:` 헤더를 보호하지 못한다. SPF/DKIM 결과를 사용자 표시 도메인과 연결하는 것은 DMARC 정렬(alignment)의 역할이다.
- 실제로 도메인을 대신해 발송하는 ESP·릴레이 집합과 동기화되어 있어야 한다. 오래된 SPF 레코드는 대표적인 도달률 문제의 원인이다.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
