---
id: 019e8d4a-1d3c-74db-bce6-9b0f67a21dbc
name: Diffie-Hellman Key Exchange
aliases:
  - DH
  - DHKE
  - Diffie–Hellman
  - ECDH
updated_at: '2026-06-03T11:41:45.148Z'
summary: >-
  A cryptographic protocol that lets two parties derive a shared secret over a
  public channel without ever transmitting the secret itself.
sources:
  - 019e8d48-5843-714e-8928-d757a5983376
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Diffie-Hellman Key Exchange

## Overview
Diffie-Hellman key exchange is a protocol that allows two parties to agree on a shared symmetric key over an insecure channel without ever sending the key. Each side combines its own private value with the other's public value to compute the same shared secret.

## Notes
- **Variants**: classic DH over a finite field, and ECDH over elliptic curves (faster, smaller keys, used in modern SSH/TLS).
- **Used in SSH**: the key-exchange phase of the SSH handshake derives the session key used for symmetric encryption.
- **Forward secrecy**: ephemeral variants (DHE/ECDHE) generate fresh keys per session, so compromise of long-term keys does not retroactively decrypt past sessions.
- **Not authentication**: DH alone does not authenticate the peer; it must be combined with signatures or host keys to prevent man-in-the-middle.

---

## 한국어

### 개요
Diffie-Hellman 키 교환은 안전하지 않은 채널에서도 비밀 키 자체를 전송하지 않고 양측이 동일한 대칭 키를 합의할 수 있게 하는 프로토콜이다. 각자 자신의 개인값과 상대의 공개값을 결합해 동일한 공유 비밀을 계산한다.

### 노트
- **변형**: 유한체 위의 고전 DH, 그리고 더 빠르고 키 길이가 짧은 타원곡선 기반 ECDH(현대 SSH/TLS에서 사용).
- **SSH에서의 사용**: SSH 핸드셰이크의 키 교환 단계에서 대칭 암호화에 쓸 세션 키를 도출한다.
- **순방향 비밀성(Forward Secrecy)**: 일회성 변형(DHE/ECDHE)을 사용하면 세션마다 새로운 키가 생성되므로 장기 키가 유출되어도 과거 세션을 복호화할 수 없다.
- **인증과 분리**: DH 자체는 상대를 인증하지 않으므로, 중간자 공격을 막으려면 서명이나 호스트 키와 함께 사용해야 한다.

## Sources

- [[raw/conversations/019e8d48-5843-714e-8928-d757a5983376|019e8d48-5843-714e-8928-d757a5983376]]
