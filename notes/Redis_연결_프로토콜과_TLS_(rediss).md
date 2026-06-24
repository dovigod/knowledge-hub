---
id: 019e8daf-6a80-754b-9e79-232dacdc25d6
title: 'Redis 연결 프로토콜과 TLS (rediss://)'
topics:
  - redis
  - tls
  - 네트워크
  - 보안
sources:
  - 019e8daf-09b5-7403-af7c-3149b422f8c2
created_at: '2026-06-03T13:32:24.064Z'
updated_at: '2026-06-03T13:32:24.064Z'
---
## TL;DR

`rediss://` is the connection scheme for Redis over TLS (SSL) encryption.

## Scheme comparison

- `redis://`
  → plaintext TCP connection (default port 6379)

- `rediss://`
  → TLS-encrypted connection (typically port 6380 or a configured TLS port)

## URL format

The general format is:

```
rediss://[:password@]host:port/db
```

Example:

```
rediss://user:password@redis.example.com:6380/0
```

## Key differences

- Whether TLS is applied at the transport layer
- Certificate-based security (defense against [[MITM]])
- Practically required in cloud environments (e.g. managed [[Redis]])

## Caveats

1. Client library support required
   Some [[Redis]] clients need TLS options to be enabled explicitly
   (e.g. Node.js's [[ioredis]], [[redis]], etc.)

2. Certificate verification
   In self-signed environments, options like `rejectUnauthorized` may need to be adjusted

3. Performance
   There is TLS handshake + encryption overhead, but the impact is generally limited relative to network cost

## Summary

`rediss://` is not just a URL scheme — think of it as an "explicit signal that forces a TLS connection."

## References

- [[Redis]] official documentation (TLS Support): https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/
- RFC 3986 (URI scheme general spec)

---

## 한국어

### TL;DR

`rediss://`는 [[Redis]]에서 TLS(SSL) 암호화를 사용하는 연결 프로토콜을 의미한다.

### 스킴 비교

- `redis://`
  → 평문 TCP 연결 (기본 포트 6379)

- `rediss://`
  → TLS로 암호화된 연결 (보통 6380 또는 설정된 TLS 포트)

### URL 형식

형식은 일반적으로 다음과 같다:

```
rediss://[:password@]host:port/db
```

예시:

```
rediss://user:password@redis.example.com:6380/0
```

### 핵심 차이

- 전송 계층에서 TLS 적용 여부
- 인증서 기반 보안 ([[MITM]] 방어)
- 클라우드 환경(예: managed [[Redis]])에서는 거의 필수

### 주의할 점

1. 클라이언트 라이브러리 지원 필요
   일부 [[Redis]] client는 TLS 옵션을 별도로 켜야 한다
   (예: Node.js의 [[ioredis]], [[redis]] 등)

2. 인증서 검증
   self-signed 환경이면 `rejectUnauthorized` 같은 옵션 조정 필요

3. 성능
   TLS handshake + 암호화 오버헤드가 있지만, 일반적으로 네트워크 비용 대비 영향은 제한적

### 정리

`rediss://`는 단순한 URL 스킴이 아니라 "TLS 연결을 강제하는 명시적 신호"라고 보면 된다.

### 참고

- [[Redis]] 공식 문서 (TLS Support): https://redis.io/docs/latest/operate/oss_and_stack/management/security/encryption/
- RFC 3986 (URI scheme general spec)
