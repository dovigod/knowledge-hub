---
id: 019e8d49-f33b-736d-82c3-7fd10d00fdfd
name: Public Key Cryptography
aliases:
  - Asymmetric Encryption
  - Public Key Cryptography
  - Public-Key Cryptography
  - asymmetric cryptography
  - asymmetric-encryption
  - public-key cryptography
  - public-key encryption
  - public-key-encryption
  - 공개키 암호
  - 공개키 암호화
  - 비대칭 암호화
updated_at: '2026-06-07T08:08:24.917Z'
summary: >-
  An asymmetric cryptosystem using a public/private key pair so anyone can
  encrypt or verify with the public key while only the private key holder can
  decrypt or sign.
sources:
  - 019e8d48-5843-714e-8928-d757a5983376
  - 019e8d8e-31db-70ae-a1a5-4073d8b737a5
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Public Key Cryptography

## Overview
Public key cryptography (asymmetric cryptography) uses a mathematically linked key pair: a public key that can be shared openly and a private key kept secret. Data encrypted with the public key can only be decrypted with the private key, and signatures produced with the private key can be verified with the public key.

> [!note] Core property
> The two keys are mathematically bound but computationally infeasible to derive one from the other — that asymmetry is what makes the public key safe to publish.

## Notes
- **Common algorithms**: RSA, ECDSA, Ed25519, Diffie-Hellman (key exchange).
- **Use in SSH**: the server stores users' public keys; the client proves possession of the matching private key during authentication without transmitting it.
- **Use in [[HTTPS]]/TLS**: certificates bind a public key to an identity, signed by a Certificate Authority. The handshake uses public-key crypto to authenticate and agree on a shared secret.
- **Tradeoff**: asymmetric operations are orders of magnitude slower than symmetric ciphers, so protocols typically use public-key crypto only for handshake/authentication and derive a symmetric session key for bulk data.
- **Key hygiene**: protect the private key with a passphrase, use modern algorithms (Ed25519 over old RSA-1024), rotate when compromise is suspected.

> [!tip] Why hybrid encryption wins
> Real-world protocols like [[HTTPS]] and SSH combine asymmetric crypto (for authenticating peers and exchanging a session key) with symmetric crypto (for bulk data transfer) — getting the trust model of the former with the speed of the latter.

> [!warning] Private key exposure is terminal
> Once a private key leaks, every past signature it produced and every session key derived through it must be considered compromised. Rotate immediately and revoke any certificates bound to it.

---

## 한국어

### 개요
공개키 암호(비대칭 암호)는 수학적으로 연결된 두 개의 키 쌍을 사용한다. 공개키는 자유롭게 공유할 수 있고, 개인키는 비밀로 보관한다. 공개키로 암호화한 데이터는 개인키로만 복호화할 수 있고, 개인키로 만든 서명은 공개키로 검증할 수 있다.

> [!note] 핵심 성질
> 두 키는 수학적으로 묶여 있지만, 한쪽에서 다른 쪽을 계산해내는 것은 사실상 불가능하다. 이 비대칭성 덕분에 공개키를 외부에 공개해도 안전하다.

### 노트
- **대표 알고리즘**: RSA, ECDSA, Ed25519, Diffie-Hellman(키 교환).
- **SSH에서의 사용**: 서버는 사용자 공개키를 저장하고, 클라이언트는 인증 시 개인키 자체를 전송하지 않고 소유 사실을 증명한다.
- **[[HTTPS]]/TLS에서의 사용**: 인증서는 공개키와 신원을 묶고, 인증 기관(CA)이 이를 서명한다. 핸드셰이크에서는 공개키 암호로 상대를 인증하고 공유 비밀을 합의한다.
- **트레이드오프**: 비대칭 연산은 대칭 암호보다 수십~수백 배 느리므로 핸드셰이크/인증에만 사용하고 실제 데이터는 대칭 세션 키로 암호화한다.
- **키 관리**: 개인키는 패스프레이즈로 보호하고, 오래된 RSA-1024 대신 Ed25519 같은 최신 알고리즘을 사용하며, 유출이 의심되면 즉시 회전한다.

> [!tip] 하이브리드 암호화가 표준이 된 이유
> [[HTTPS]], SSH 같은 실제 프로토콜은 비대칭 암호(상대 인증과 세션 키 교환)와 대칭 암호(실제 데이터 전송)를 결합한다. 전자의 신뢰 모델과 후자의 속도를 동시에 얻는 구조다.

> [!warning] 개인키 유출은 치명적
> 개인키가 한 번 유출되면 그 키로 만든 과거 서명과 그 키를 통해 합의된 모든 세션 키가 침해된 것으로 간주해야 한다. 즉시 키를 회전하고, 해당 키에 묶인 인증서는 폐기해야 한다.

## Sources

- [[raw/conversations/019e8d48-5843-714e-8928-d757a5983376|019e8d48-5843-714e-8928-d757a5983376]]
- [[raw/conversations/019e8d8e-31db-70ae-a1a5-4073d8b737a5|019e8d8e-31db-70ae-a1a5-4073d8b737a5]]
