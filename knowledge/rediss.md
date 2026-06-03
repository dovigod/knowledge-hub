---
id: 019e8db0-837e-70f2-bdb7-afed58d67861
name: rediss
aliases:
  - redis over tls
  - redis-tls
  - 'rediss://'
updated_at: '2026-06-03T13:33:35.999Z'
summary: URL scheme that signals a TLS-encrypted connection to a Redis server.
sources:
  - 019e8daf-09b5-7403-af7c-3149b422f8c2
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# rediss

## Overview

`rediss://` is the URL scheme that tells a Redis client to connect over [[TLS]] instead of plaintext TCP. It is not a separate protocol — the wire protocol is still [[Redis]]'s RESP — but the scheme is an explicit signal that the transport must be encrypted.

> [!note] Scheme comparison
> - `redis://` → plaintext TCP, default port 6379
> - `rediss://` → TLS-wrapped TCP, typically port 6380

## Notes

- Format: `rediss://[user:password@]host:port/db`
- Example: `rediss://user:password@redis.example.com:6380/0`
- Provides transport-layer encryption and certificate-based defense against MITM attacks.
- Practically required for managed [[Redis]] services in the cloud.

> [!warning] Client library support
> Not every client honors `rediss://` automatically. Some need an explicit TLS option:
> - [[ioredis]]: pass `tls: {}` (or detailed options)
> - `node-redis`: configure the `socket.tls` block

> [!tip] Self-signed certificates
> For self-signed or internal CA setups, you may need to relax verification (`rejectUnauthorized: false`) or supply a custom CA bundle.

> [!example] Usage
> ```
> rediss://user:password@redis.example.com:6380/0
> ```

References:
- Redis TLS docs: https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/
- [[RFC 3986]] (URI generic syntax)

---

## 한국어

### 개요

`rediss://`는 Redis 클라이언트에게 평문 TCP 대신 [[TLS]]로 연결하라고 알리는 URL 스킴이다. 별도의 프로토콜이 아니라 와이어 프로토콜은 여전히 [[Redis]]의 RESP이며, 스킴 자체가 "전송 계층을 암호화하라"는 명시적 신호 역할을 한다.

> [!note] 스킴 비교
> - `redis://` → 평문 TCP, 기본 포트 6379
> - `rediss://` → TLS로 감싼 TCP, 보통 6380 포트

### 노트

- 형식: `rediss://[user:password@]host:port/db`
- 예시: `rediss://user:password@redis.example.com:6380/0`
- 전송 계층 암호화 및 인증서 기반 MITM 방어를 제공한다.
- 클라우드 매니지드 [[Redis]] 환경에서는 사실상 필수다.

> [!warning] 클라이언트 라이브러리 지원
> 모든 클라이언트가 `rediss://`를 자동 처리하지는 않는다. 일부는 TLS 옵션을 별도로 켜야 한다:
> - [[ioredis]]: `tls: {}` (또는 세부 옵션) 전달
> - `node-redis`: `socket.tls` 블록 설정

> [!tip] 자체 서명 인증서
> self-signed 또는 사내 CA 환경에서는 검증을 완화(`rejectUnauthorized: false`)하거나 커스텀 CA 번들을 제공해야 할 수 있다.

> [!example] 사용 예
> ```
> rediss://user:password@redis.example.com:6380/0
> ```

참고:
- Redis TLS 문서: https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/
- [[RFC 3986]] (URI 일반 구문)

## Sources

- [[raw/conversations/019e8daf-09b5-7403-af7c-3149b422f8c2|019e8daf-09b5-7403-af7c-3149b422f8c2]]
