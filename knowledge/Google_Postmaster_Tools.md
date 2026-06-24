---
id: 019e8d29-f07f-75fc-ada7-64e27bb80196
name: Google Postmaster Tools
aliases:
  - Gmail Postmaster Tools
  - Postmaster Tools
updated_at: '2026-06-03T11:06:36.543Z'
summary: >-
  Google's free dashboard that exposes Gmail-side reputation, authentication,
  spam rate, and delivery-error metrics for domains that send to Gmail users.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Google Postmaster Tools

## Overview

Google Postmaster Tools is a free dashboard provided by Google that exposes Gmail-side delivery metrics — domain and IP reputation, spam-complaint rate, authentication pass rates (SPF/DKIM/DMARC), encryption, and delivery errors — for domains that send mail to Gmail users. It is the canonical observability surface for diagnosing Gmail deliverability.

## Notes

- Access requires DNS-based ownership verification of the sending domain.
- Reports the domain's reputation as Bad / Low / Medium / High and shows the spam rate Gmail users perceive.
- Considered essentially mandatory for any sender targeting meaningful Gmail volume, particularly after the 2024 Gmail bulk-sender rules.
- Complements Microsoft SNDS (Outlook/Hotmail side) and third-party tools such as Sender Score and MXToolbox.

---

## 한국어

### 개요

Google Postmaster Tools는 Gmail 사용자에게 메일을 보내는 도메인에 대해 도메인·IP 평판, 스팸 신고율, 인증(SPF/DKIM/DMARC) 통과율, 암호화, 전송 오류 등 Gmail 측의 도달률 지표를 노출하는 Google의 무료 대시보드다. Gmail 도달률 문제를 진단할 때 사실상 표준이 되는 관측 도구다.

### 노트

- 사용하려면 발신 도메인에 대한 DNS 기반 소유권 검증이 필요하다.
- 도메인 평판을 Bad / Low / Medium / High로 보여주며, Gmail 사용자 입장의 스팸 신고율을 확인할 수 있다.
- 2024년 Gmail 대량발송자 정책 이후, Gmail로 의미 있는 볼륨을 발송하는 모든 발신자에게 사실상 필수 도구로 간주된다.
- Outlook·Hotmail 측의 Microsoft SNDS, Sender Score·MXToolbox 같은 서드파티 도구와 보완적으로 사용한다.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
