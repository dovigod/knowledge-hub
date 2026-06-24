---
id: 019ea11b-0808-710f-b3a1-81938c3ff5fe
title: SSH 프로토콜 개요와 연결 과정
topics:
  - ssh
  - 보안
  - 원격 접속
  - 공개키 인증
  - 암호화
  - 네트워크 프로토콜
sources:
  - 019e8d48-5843-714e-8928-d757a5983376
  - 019e8d52-83e5-72d1-a0b2-281e58439695
created_at: '2026-06-07T08:02:43.848Z'
updated_at: '2026-06-07T08:04:06.816Z'
---
## Overview

SSH (Secure Shell) is a protocol for securely connecting to and executing commands on remote servers over a network. It provides an encrypted channel over an unsecured network, replacing older, insecure protocols like Telnet, rlogin, and rsh.

## Key Features

- **Encrypted communication**: All traffic between client and server is encrypted, protecting against eavesdropping and man-in-the-middle attacks.
- **Authentication**: Supports multiple authentication methods (password, public key, host-based, keyboard-interactive).
- **Integrity**: Uses MACs (Message Authentication Codes) to detect tampering.
- **Port forwarding / tunneling**: Can tunnel arbitrary TCP connections, X11 sessions, and create SOCKS proxies.
- **File transfer**: Bundled protocols like [[SCP]] and [[SFTP]] ride on top of SSH.
- **Default port**: TCP 22.

## Connection Process

The SSH handshake proceeds in three main phases:

### 1. Server Authentication (Host Key Verification)

- The client connects to the server on port 22.
- The server presents its **host public key**.
- The client checks this key against its `~/.ssh/known_hosts` file.
  - If the key is unknown, the user is prompted to accept it (TOFU — Trust On First Use).
  - If the key has changed, the connection is refused (possible MITM attack).
- This step authenticates the **server** to the **client**.

### 2. Key Exchange (Session Key Establishment)

- Client and server negotiate a **session key** using a key exchange algorithm such as [[Diffie-Hellman]] or [[ECDH]] (Elliptic-Curve Diffie-Hellman).
- The session key is a **symmetric key** used to encrypt all subsequent traffic.
- Symmetric encryption (e.g., AES, ChaCha20) is used because it is much faster than asymmetric encryption for bulk data.
- The key exchange itself is performed using asymmetric cryptography so that the session key is never transmitted in the clear.

### 3. User Authentication

After the encrypted channel is established, the user must authenticate. Common methods:

- **Password authentication**: User sends a password over the encrypted channel.
- **Public key authentication** (preferred):
  - User has a key pair (`~/.ssh/id_rsa` private, `~/.ssh/id_rsa.pub` public).
  - The public key is placed in the server's `~/.ssh/authorized_keys`.
  - The server sends a challenge; the client signs it with the private key; the server verifies the signature using the stored public key.
- **Keyboard-interactive / [[MFA]]**: Used with [[PAM]], 2FA tokens, etc.

## Public Key Authentication in Detail

```
Client                                Server
------                                ------
                                      ~/.ssh/authorized_keys
                                        contains client's public key

1. "I want to log in as user X with key K"  ──>
                                      <──  random challenge (nonce)
2. sign(nonce, private_key)             ──>
                                      verify(signature, public_key)
                                      <──  OK → session begins
```

The **private key never leaves the client**. Only a signature derived from it is sent, and that signature is bound to the challenge, so it cannot be replayed.

## Symmetric vs. Asymmetric Encryption in SSH

| Stage | Cryptography | Why |
|---|---|---|
| Host key verification | Asymmetric | Identify the server |
| Key exchange | Asymmetric (DH/ECDH) | Establish shared secret without transmitting it |
| Bulk data transfer | Symmetric (AES, ChaCha20) | Fast, low CPU overhead |
| Integrity check | MAC (HMAC-SHA2, etc.) | Detect tampering |

## SSH vs. HTTPS

Both use a similar TLS-like pattern (asymmetric for setup, symmetric for bulk), but:

- **HTTPS** authenticates servers via a [[Certificate Authority]] (CA) chain. Trust is delegated to public CAs.
- **SSH** typically uses **TOFU** (Trust On First Use) for host keys, or organization-managed CA-signed host certificates. There is no global CA hierarchy by default.
- HTTPS is request/response (HTTP); SSH is an interactive, bidirectional, multiplexed channel suitable for shells, file transfer, and tunneling.

## Operational Configuration Examples

### Client-side `~/.ssh/config`

```
Host prod
    HostName prod.example.com
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent no
```

### Server-side `/etc/ssh/sshd_config` (hardening)

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
AllowUsers deploy
Port 22
```

Common hardening practices:

- Disable root login and password authentication.
- Use Ed25519 or RSA ≥ 3072-bit keys.
- Restrict allowed users / groups.
- Run on a non-default port (minor obscurity, not security).
- Use [[fail2ban]] or similar to throttle brute-force attempts.
- Rotate host keys and user keys periodically.

---

## 한국어

### 개요

SSH(Secure Shell)는 네트워크를 통해 원격 서버에 안전하게 접속하고 명령을 실행하기 위한 프로토콜이다. 안전하지 않은 네트워크 위에 암호화된 채널을 제공하며, Telnet, rlogin, rsh 같은 구식의 안전하지 않은 프로토콜을 대체한다.

### 주요 기능

- **암호화된 통신**: 클라이언트와 서버 간의 모든 트래픽이 암호화되어 도청 및 중간자 공격(MITM)으로부터 보호된다.
- **인증**: 다양한 인증 방식(비밀번호, 공개키, 호스트 기반, 키보드 인터랙티브)을 지원한다.
- **무결성**: MAC(Message Authentication Code)을 사용하여 변조를 탐지한다.
- **포트 포워딩 / 터널링**: 임의의 TCP 연결, X11 세션을 터널링할 수 있고, SOCKS 프록시도 만들 수 있다.
- **파일 전송**: [[SCP]], [[SFTP]] 같은 번들 프로토콜이 SSH 위에서 동작한다.
- **기본 포트**: TCP 22.

### 연결 과정

SSH 핸드셰이크는 세 가지 주요 단계로 진행된다:

#### 1. 서버 인증 (호스트 키 검증)

- 클라이언트가 서버의 22번 포트로 접속한다.
- 서버가 자신의 **호스트 공개키**를 제시한다.
- 클라이언트는 이 키를 `~/.ssh/known_hosts` 파일과 대조한다.
  - 키가 알려져 있지 않으면 사용자에게 수락 여부를 묻는다 (TOFU — Trust On First Use).
  - 키가 변경되었다면 연결이 거부된다 (MITM 공격 가능성).
- 이 단계는 **서버**를 **클라이언트**에게 인증하는 과정이다.

#### 2. 키 교환 (세션 키 수립)

- 클라이언트와 서버가 [[Diffie-Hellman]] 또는 [[ECDH]](타원곡선 디피-헬만) 같은 키 교환 알고리즘을 사용해 **세션 키**를 협상한다.
- 세션 키는 이후의 모든 트래픽을 암호화하는 데 사용되는 **대칭키**이다.
- 대량 데이터에 대해 비대칭 암호화보다 훨씬 빠르기 때문에 대칭 암호화(AES, ChaCha20 등)를 사용한다.
- 키 교환 자체는 비대칭 암호를 사용하므로 세션 키가 평문으로 전송되는 일은 없다.

#### 3. 사용자 인증

암호화된 채널이 수립된 후 사용자 인증이 진행된다. 일반적인 방법:

- **비밀번호 인증**: 사용자가 암호화된 채널을 통해 비밀번호를 전송한다.
- **공개키 인증** (권장):
  - 사용자가 키 쌍을 가진다 (`~/.ssh/id_rsa` 개인키, `~/.ssh/id_rsa.pub` 공개키).
  - 공개키를 서버의 `~/.ssh/authorized_keys`에 등록한다.
  - 서버가 챌린지를 보내면, 클라이언트는 개인키로 서명하고, 서버는 저장된 공개키로 서명을 검증한다.
- **키보드 인터랙티브 / [[MFA]]**: [[PAM]], 2FA 토큰 등과 함께 사용된다.

### 공개키 인증 상세

```
Client                                Server
------                                ------
                                      ~/.ssh/authorized_keys
                                        contains client's public key

1. "I want to log in as user X with key K"  ──>
                                      <──  random challenge (nonce)
2. sign(nonce, private_key)             ──>
                                      verify(signature, public_key)
                                      <──  OK → session begins
```

**개인키는 절대 클라이언트를 벗어나지 않는다.** 단지 개인키로부터 파생된 서명만 전송되며, 그 서명은 챌린지에 종속되므로 재사용(리플레이)이 불가능하다.

### SSH에서의 대칭 vs. 비대칭 암호화

| 단계 | 암호화 방식 | 이유 |
|---|---|---|
| 호스트 키 검증 | 비대칭 | 서버 신원 확인 |
| 키 교환 | 비대칭 (DH/ECDH) | 공유 비밀을 전송 없이 수립 |
| 대량 데이터 전송 | 대칭 (AES, ChaCha20) | 빠르고 CPU 부하가 적음 |
| 무결성 검사 | MAC (HMAC-SHA2 등) | 변조 탐지 |

### SSH vs. HTTPS

둘 다 TLS와 비슷한 패턴(수립은 비대칭, 대량 전송은 대칭)을 사용하지만:

- **HTTPS**는 [[Certificate Authority]](CA) 체인을 통해 서버를 인증한다. 신뢰가 공인 CA에게 위임된다.
- **SSH**는 호스트 키에 대해 일반적으로 **TOFU**(Trust On First Use)를 사용하거나, 조직에서 관리하는 CA 서명 호스트 인증서를 사용한다. 기본적으로 전 세계적인 CA 계층 구조가 없다.
- HTTPS는 요청/응답(HTTP) 기반이지만, SSH는 셸, 파일 전송, 터널링에 적합한 인터랙티브, 양방향, 멀티플렉싱 채널이다.

### 운영 환경 설정 예시

#### 클라이언트 측 `~/.ssh/config`

```
Host prod
    HostName prod.example.com
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent no
```

#### 서버 측 `/etc/ssh/sshd_config` (하드닝)

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
AllowUsers deploy
Port 22
```

일반적인 하드닝 관행:

- 루트 로그인과 비밀번호 인증을 비활성화한다.
- Ed25519 또는 3072비트 이상의 RSA 키를 사용한다.
- 허용된 사용자/그룹을 제한한다.
- 기본 포트가 아닌 다른 포트에서 운영한다 (보안이라기보단 약간의 모호화).
- [[fail2ban]] 등을 사용해 무차별 대입 시도를 차단한다.
- 호스트 키와 사용자 키를 주기적으로 교체한다.
