---
id: 019e8d29-aefc-71df-91f0-c39febd0a837
name: IP Warming
aliases:
  - Email IP Warming
  - IP Warm-up
  - IP 워밍업
updated_at: '2026-06-03T11:06:19.772Z'
summary: >-
  Practice of gradually ramping outbound email volume from a new sending IP so
  that receiving ISPs build a positive reputation instead of blocking it.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# IP Warming

## Overview

IP warming is the practice of gradually increasing the email volume sent from a new IP address so that receiving ISPs can establish a positive reputation for it. Starting at full volume on a cold IP almost always triggers immediate filtering or blocking.

## Notes

- Typical curve: day 1 sends 50–100 messages; double the daily volume every few days; reach the target volume over roughly 4–8 weeks.
- During warm-up, prioritize the most engaged recipients (recent opens/clicks) so positive engagement signals dominate early reputation data.
- AWS SES and several other providers offer automated warm-up that throttles outbound volume on your behalf.
- Pairs with authentication setup (SPF / DKIM / DMARC) and list hygiene; warming a clean list on an unauthenticated domain still fails.

---

## 한국어

### 개요

IP 워밍업은 새로 사용하는 발신 IP의 이메일 발송량을 점진적으로 늘려서 수신 ISP가 해당 IP에 긍정적인 평판을 쌓을 수 있도록 하는 절차다. 평판이 없는 새 IP에서 처음부터 대량 발송하면 거의 즉시 필터링되거나 차단된다.

### 노트

- 일반적인 곡선: 첫날 50~100통 발송, 며칠마다 일일 발송량을 2배씩 증가, 4~8주에 걸쳐 목표 볼륨 도달.
- 워밍업 기간에는 최근 열람·클릭이 많은 engagement 높은 수신자에게 먼저 발송하여 초기 평판 데이터를 긍정적인 신호로 채운다.
- AWS SES를 비롯한 일부 서비스는 발송량을 자동으로 조절해 주는 자동 워밍업을 제공한다.
- 인증 설정(SPF / DKIM / DMARC) 및 리스트 위생과 함께 적용되어야 하며, 인증이 미흡한 도메인에서는 깨끗한 리스���로 워밍업해도 효과가 없다.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
