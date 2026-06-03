---
id: 019e8d29-e26a-701d-bab8-15a6ecdc1045
name: DMARC
aliases:
  - 'Domain-based Message Authentication, Reporting and Conformance'
updated_at: '2026-06-03T11:06:32.938Z'
summary: >-
  DNS-published policy that ties SPF and DKIM results to the visible From-domain
  and tells receivers what to do when authentication fails, plus where to send
  aggregate reports.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# DMARC

## Overview

DMARC (Domain-based Message Authentication, Reporting and Conformance) is a DNS-published policy layered on top of SPF and DKIM. It tells receiving servers how to align SPF/DKIM results with the visible `From:` domain, what action to take when authentication fails (`none`, `quarantine`, `reject`), and where to deliver aggregate and forensic reports.

## Notes

- Published as a TXT record at `_dmarc.<domain>`.
- Policy levels: `p=none` (monitor only), `p=quarantine` (suspicious), `p=reject` (block).
- Gmail and Yahoo bulk-sender rules (effective 2024) require at minimum a published DMARC policy of `p=none` for senders above the bulk threshold.
- Aggregate (`rua=`) reports are the practical entry point to observing real-world authentication outcomes for your domain.
- Alignment is what makes DMARC meaningful: an SPF or DKIM pass on an unrelated domain does not satisfy DMARC unless it aligns with the From-domain.

---

## 한국어

### 개요

DMARC(Domain-based Message Authentication, Reporting and Conformance)는 SPF와 DKIM 위에 얹는 DNS 게시 정책이다. 수신 서버에 SPF/DKIM 결과를 사용자에게 보이는 `From:` 도메인과 어떻게 정렬할지, 인증 실패 시 어떤 조치를 취할지(`none`, `quarantine`, `reject`), 집계·포렌식 리포트를 어디로 보낼지를 알려준다.

### 노트

- `_dmarc.<도메인>` 위치에 TXT 레코드로 게시한다.
- 정책 수준: `p=none`(모니터링 전용), `p=quarantine`(의심), `p=reject`(차단).
- 2024년부터 적용된 Gmail·Yahoo 대량발송자 정책은 일정 발송량 이상의 발신자에 대해 최소 `p=none` 수준의 DMARC 정책 게시를 요구한다.
- 집계 리포트(`rua=`)는 도메인의 실제 인증 결과를 관측할 수 있는 실용적 출발점이다.
- DMARC의 핵심은 정렬(alignment)이다. 무관한 도메인에서의 SPF/DKIM pass는 From 도메인과 정렬되지 않으면 DMARC를 만족시키지 못한다.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
