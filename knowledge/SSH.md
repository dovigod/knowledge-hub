---
id: 019e8d49-f338-77ce-8af9-a7932878e79f
name: SSH
aliases:
  - SSH
  - Secure Shell
  - ssh
updated_at: '2026-06-03T11:52:29.315Z'
summary: >-
  A cryptographic network protocol for securely accessing and operating remote
  servers over an untrusted network.
sources:
  - 019e8d48-5843-714e-8928-d757a5983376
  - 019e8d52-83e5-72d1-a0b2-281e58439695
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# SSH

## Overview
SSH (Secure Shell) is a network protocol that enables secure remote login, command execution, and file transfer over an untrusted network. It establishes an encrypted channel between a client and a server, protecting against eavesdropping, connection hijacking, and other network-level attacks.

## Notes
- **Connection flow**: TCP handshake → server authentication (host key verification) → key exchange (e.g. Diffie-Hellman) → user authentication → encrypted session.
- **Authentication methods**: password, public key (most common in production), keyboard-interactive, GSSAPI.
- **Public key authentication**: client holds a private key; server stores the matching public key in `~/.ssh/authorized_keys`. The server challenges the client to prove possession of the private key without transmitting it.
- **Symmetric encryption**: after key exchange, both sides derive a shared session key and use a symmetric cipher (AES, ChaCha20) for bulk data — faster than asymmetric crypto.
- **vs HTTPS**: HTTPS authenticates the server via a CA-signed certificate chain and is request/response oriented; SSH typically uses TOFU (trust-on-first-use) host keys stored in `known_hosts` and provides a long-lived interactive or tunneled session.
- **Common uses**: remote shell, `scp`/`sftp` file transfer, port forwarding (tunneling), git over SSH, `ProxyJump` bastion hosts.
- **Operational hardening**: disable password auth, disable root login, restrict users, use ED25519 keys, enable fail2ban, change default port only as defense-in-depth.
- **Layer positioning**: SSH operates at the application layer (L7) over TCP, but is often discussed alongside L3/L4/L7 network security since it provides transport-level encryption similar in spirit to TLS. Related concepts: L3 security (IPsec, network ACLs, firewall rules at IP level), L4 security (TCP/UDP port filtering, stateful firewalls), and L7 security (application-aware policies as in [[Cilium]]).

---

## 한국어

### 개요
SSH(Secure Shell)는 신뢰할 수 없는 네트워크에서 원격 로그인, 명령 실행, 파일 전송을 안전하게 수행하기 위한 네트워크 프로토콜이다. 클라이언트와 서버 사이에 암호화된 채널을 만들어 도청, 연결 가로채기 등 네트워크 수준의 공격으로부터 통신을 보호한다.

### 노트
- **연결 과정**: TCP 핸드셰이크 → 서버 인증(호스트 키 검증) → 키 교환(예: Diffie-Hellman) → 사용자 인증 → 암호화 세션 시작.
- **인증 방식**: 비밀번호, 공개키(운영 환경에서 가장 많이 사용), 키보드 인터랙티브, GSSAPI.
- **공개키 인증**: 클라이언트가 개인키를 보관하고, 서버는 대응하는 공개키를 `~/.ssh/authorized_keys`에 저장한다. 서버는 개인키 자체를 전송하지 않으면서 소유 여부를 챌린지로 검증한다.
- **대칭키 암호화**: 키 교환 이후 양측이 공유 세션 키를 도출하고 AES, ChaCha20 같은 대칭 암호로 실제 데이터를 암호화한다. 비대칭 방식보다 훨씬 빠르다.
- **HTTPS와의 차이**: HTTPS는 CA가 서명한 인증서 체인으로 서버를 인증하고 요청/응답 중심으로 동작한다. SSH는 보통 `known_hosts`에 저장된 호스트 키 기반의 TOFU(처음 연결 시 신뢰) 방식이며, 장시간 유지되는 인터랙티브 세션이나 터널을 제공한다.
- **주요 용도**: 원격 셸, `scp`/`sftp` 파일 전송, 포트 포워딩(터널링), git over SSH, `ProxyJump`를 활용한 배스천 호스트 접근.
- **운영 환경 강화**: 비밀번호 인증 비활성화, root 로그인 차단, 사용자 제한, ED25519 키 사용, fail2ban 적용, 기본 포트 변경은 보조적 방어 수단으로 활용.
- **계층 관점**: SSH는 TCP 위에서 동작하는 애플리케이션 계층(L7) 프로토콜이지만, TLS와 비슷하게 전송 구간 암호화를 제공하기 때문에 L3/L4/L7 네트워크 보안과 함께 논의되는 경우가 많다. 관련 개념으로는 L3 보안(IPsec, 네트워크 ACL, IP 단위 방화벽 규칙), L4 보안(TCP/UDP 포트 필터링, 스테이트풀 방화벽), L7 보안([[Cilium]]처럼 애플리케이션을 이해하는 정책)이 있다.

## Sources

- [[raw/conversations/019e8d48-5843-714e-8928-d757a5983376|019e8d48-5843-714e-8928-d757a5983376]]
- [[raw/conversations/019e8d52-83e5-72d1-a0b2-281e58439695|019e8d52-83e5-72d1-a0b2-281e58439695]]
