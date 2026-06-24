---
id: 019e8d29-d30c-7786-9ebb-f0e13264c722
name: DKIM
aliases:
  - DomainKeys Identified Mail
updated_at: '2026-06-03T11:06:29.004Z'
summary: >-
  Email authentication standard that signs outgoing messages with a
  domain-controlled cryptographic key so receivers can verify the message was
  not altered and was sent by an authorized server.
sources:
  - 019e8d28-0dc9-755c-834d-aad29d26a380
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# DKIM

## Overview

DKIM (DomainKeys Identified Mail) is an email authentication standard in which the sending mail server signs selected headers and the body of each message with a private key. The corresponding public key is published in DNS under a selector for the signing domain, allowing receivers to verify the signature and detect tampering or unauthorized senders.

## Notes

- Public key is published at `<selector>._domainkey.<domain>` as a DNS TXT record.
- DKIM survives forwarding better than SPF because it travels with the message rather than depending on the connecting IP.
- Together with SPF and DMARC, DKIM is part of the baseline authentication stack required by Gmail / Yahoo bulk-sender rules in effect since 2024.
- Key rotation is part of normal hygiene; long-lived keys with weak sizes (<1024 bits) should be replaced.

---

## 한국어

### 개요

DKIM(DomainKeys Identified Mail)은 발신 메일 서버가 도메인이 보유한 비공개 키로 선택된 헤더와 본문에 서명하는 이메일 인증 표준이다. 대응하는 공개 키는 서명 도메인의 selector 아래에 DNS TXT 레코드로 게시되며, 수신 서버는 이를 통해 서명을 검증하고 위·변조 또는 무단 발송을 탐지한다.

### 노트

- 공개 키는 `<selector>._domainkey.<도메인>` 위치에 DNS TXT 레코드로 게시한다.
- DKIM은 메시지 자체에 서명을 담아 전달하므로, 연결 IP에 의존하는 SPF에 비해 포워딩 상황에서 더 잘 살아남는다.
- SPF, DMARC와 함께 2024년부터 발효된 Gmail·Yahoo 대량발송자 정책의 기본 인증 스택을 구성한다.
- 키 로테이션은 일상적인 보안 위생이며, 키 길이가 약한(<1024비트) 장기 보존 키는 교체해야 한다.

## Sources

- [[raw/conversations/019e8d28-0dc9-755c-834d-aad29d26a380|019e8d28-0dc9-755c-834d-aad29d26a380]]
