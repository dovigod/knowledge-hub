---
id: 019e8d4a-0bbb-70dc-be31-15d66da27a0e
name: Symmetric Encryption
aliases:
  - symmetric-key encryption
  - 대칭키 암호화
updated_at: '2026-06-03T11:41:40.667Z'
summary: >-
  An encryption scheme in which the same secret key is used for both encryption
  and decryption, optimized for fast bulk data protection.
sources:
  - 019e8d48-5843-714e-8928-d757a5983376
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Symmetric Encryption

## Overview
Symmetric encryption uses a single shared secret key for both encrypting and decrypting data. It is much faster than asymmetric cryptography and is the workhorse for bulk data confidentiality once two parties have securely agreed on a key.

## Notes
- **Common algorithms**: AES (128/256-bit), ChaCha20, 3DES (legacy).
- **Modes**: GCM and ChaCha20-Poly1305 provide authenticated encryption (confidentiality + integrity); CBC alone does not.
- **Key distribution problem**: both parties must obtain the same secret without exposing it — usually solved by an asymmetric key-exchange step (Diffie-Hellman, RSA key wrap).
- **Use in SSH/TLS**: after the handshake derives a session key, all subsequent traffic is encrypted symmetrically for performance.

---

## 한국어

### 개요
대칭키 암호화는 암호화와 복호화에 동일한 비밀 키를 사용하는 방식이다. 비대칭 암호보다 훨씬 빠르며, 두 당사자가 안전하게 키를 공유한 뒤에는 대량 데이터의 기밀성을 보호하는 주력 수단으로 쓰인다.

### 노트
- **대표 알고리즘**: AES(128/256비트), ChaCha20, 3DES(레거시).
- **운용 모드**: GCM, ChaCha20-Poly1305은 인증된 암호화(기밀성+무결성)를 제공한다. CBC 단독으로는 무결성을 보장하지 않는다.
- **키 배포 문제**: 양측이 동일한 비밀 키를 노출 없이 공유해야 한다. 보통 Diffie-Hellman, RSA 키 래핑 같은 비대칭 키 교환 단계로 해결한다.
- **SSH/TLS에서의 사용**: 핸드셰이크로 세션 키를 도출한 후 모든 트래픽은 성능을 위해 대칭 암호로 보호된다.

## Sources

- [[raw/conversations/019e8d48-5843-714e-8928-d757a5983376|019e8d48-5843-714e-8928-d757a5983376]]
