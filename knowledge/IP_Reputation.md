---
id: 019e8d29-91e2-737c-878f-cacdff3bf4e9
name: IP Reputation
aliases:
  - Sender IP Reputation
  - 발신 IP 평판
updated_at: '2026-06-03T11:06:12.323Z'
summary: >-
  Trust score that receiving mail servers assign to a sending IP address, used
  to decide whether to deliver, spam-folder, or reject inbound email.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# IP Reputation

## Overview

IP reputation is the trust score that receiving mail servers (Gmail, Outlook, etc.) assign to a sending IP. It is the primary signal that determines whether mail is delivered to the inbox, routed to spam, or rejected outright.

## Notes

Evaluation factors:
- **Complaint rate** — fraction of recipients clicking "Report spam". >0.1% is risky; Gmail effectively blocks above 0.3%.
- **Hard bounce rate** — sending to non-existent addresses signals poor list hygiene.
- **Spam trap hits** — hitting ISP-planted decoy addresses crashes reputation immediately.
- **Volume pattern** — sudden spikes from a previously quiet IP trigger filtering.
- **Authentication** — SPF / DKIM / DMARC alignment.
- **Blacklist listings** — Spamhaus, Barracuda, and other RBLs.
- **Engagement** — opens and replies vs. deletes without reading.

Shared vs. dedicated IP:
- **Shared** — reputation pooled with other senders; suits low volume (<tens of thousands/month).
- **Dedicated** — reputation is solely yours; needs steady high volume (100k+/month) to establish a signal.

IP warming: a new IP has no reputation. Start with ~50–100 messages/day, double every few days over 4–8 weeks, prioritize highly engaged recipients first. AWS SES offers automated warm-up.

Monitoring tools:
- Google Postmaster Tools (Gmail)
- Microsoft SNDS (Outlook/Hotmail)
- Sender Score by Validity (0–100, 80+ is healthy)
- MXToolbox Blacklist Check
- Spamhaus Lookup

Since 2024, **domain reputation** has overtaken IP reputation in importance due to Gmail/Yahoo bulk sender rules. Implications: SPF + DKIM + DMARC (`p=none` minimum) are mandatory; separate marketing traffic onto a subdomain (e.g. `marketing.example.com`) to protect transactional mail; the `List-Unsubscribe` one-click header is required.

---

## 한국어

### 개요

IP 평판(IP Reputation)은 수신 측 메일 서버(Gmail, Outlook 등)가 발신 IP에 부여하는 신뢰도 점수다. 메일을 인박스로 보낼지, 스팸함으로 보낼지, 아예 거부할지를 결정하는 핵심 기준이다.

### 노트

평가 요소:
- **스팸 신고율(complaint rate)** — 수신자가 "스팸 신고"를 누르는 비율. 0.1% 이상이면 위험, Gmail은 0.3% 넘으면 사실상 차단.
- **하드 바운스율** — 존재하지 않는 주소로 발송하는 비율. 리스트 품질이 나쁘다는 신호.
- **스팸트랩 적중** — ISP가 심어놓은 미끼 주소로 발송하면 즉시 평판 급락.
- **발송량 패턴** — 평소 잠잠하던 IP가 갑자기 대량 발송하면 필터링 대상.
- **인증 여부** — SPF / DKIM / DMARC 정합성.
- **블랙리스트 등재** — Spamhaus, Barracuda 등 RBL.
- **수신자 반응(engagement)** — 열람·답장 vs. 읽지 않고 삭제.

공유 IP vs. 전용 IP:
- **공유 IP** — 같은 IP를 쓰는 다른 발송자들과 평판을 공유. 소량 발송(월 수만 통 이하)에 적합.
- **전용 IP** — 내 발송 품질이 곧 내 평판. 월 10만+ 통의 꾸준한 볼륨이 있어야 의미가 있다.

IP 워밍업: 새 IP는 평판이 없으므로 첫날 50~100통으로 시작해 며칠마다 2배씩 증가, 4~8주에 걸쳐 목표 볼륨에 도달한다. 워밍업 초기에는 engagement 높은 수신자에게 먼저 발송. AWS SES는 자동 워밍업을 제공한다.

모니터링 도구:
- Google Postmaster Tools (Gmail)
- Microsoft SNDS (Outlook/Hotmail)
- Sender Score by Validity (0~100, 80+ 양호)
- MXToolbox Blacklist Check
- Spamhaus Lookup

2024년 이후 Gmail/Yahoo 대량발송자 정책 강화로 IP 평판보다 **도메인 평판**의 비중이 커졌다. 그 결과 SPF + DKIM + DMARC(`p=none` 이상) 설정이 필수, 마케팅 트래픽은 `marketing.example.com` 같은 서브도메인으로 분리해 트랜잭션 메일 평판을 보호하며, 원클릭 `List-Unsubscribe` 헤더가 요구된다.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
