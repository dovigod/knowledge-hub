---
id: 019e8d4a-4474-72b8-8129-489a8142fd27
name: SSH Tunneling
aliases:
  - SSH port forwarding
  - port forwarding
  - ssh tunnel
updated_at: '2026-06-03T11:41:55.188Z'
summary: >-
  Using an SSH connection to forward arbitrary TCP traffic through an encrypted
  channel, commonly to reach private services or bypass network restrictions.
sources:
  - 019e8d48-5843-714e-8928-d757a5983376
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# SSH Tunneling

## Overview
SSH tunneling wraps arbitrary TCP traffic inside an existing SSH connection, turning SSH into a general-purpose encrypted transport. It is commonly used to reach services on a private network through a bastion host, to add encryption to plaintext protocols, or to bypass firewall restrictions.

## Notes
- **Local forwarding (`-L`)**: forwards a local port to a remote target via the SSH server (`ssh -L 5432:db.internal:5432 bastion`).
- **Remote forwarding (`-R`)**: exposes a local port on the remote server (`ssh -R 8080:localhost:3000 server`).
- **Dynamic forwarding (`-D`)**: turns SSH into a SOCKS proxy for ad-hoc routing.
- **ProxyJump (`-J`)**: chains through one or more bastion hosts without needing manual nested SSH sessions.
- **Security considerations**: tunneling can bypass network controls; servers can restrict it via `AllowTcpForwarding`, `PermitOpen`, and per-user configuration.

---

## 한국어

### 개요
SSH 터널링은 기존 SSH 연결 위에 임의의 TCP 트래픽을 캡슐화해 흘려보내는 기법으로, SSH를 범용 암호화 전송 수단으로 활용한다. 사설망 안의 서비스에 배스천 호스트를 통해 접근하거나, 평문 프로토콜에 암호화를 더하거나, 방화벽 제약을 우회할 때 자주 쓰인다.

### 노트
- **로컬 포워딩(`-L`)**: 로컬 포트를 SSH 서버를 통해 원격 대상으로 전달한다(`ssh -L 5432:db.internal:5432 bastion`).
- **원격 포워딩(`-R`)**: 로컬 포트를 원격 서버에 노출시킨다(`ssh -R 8080:localhost:3000 server`).
- **동적 포워딩(`-D`)**: SSH를 SOCKS 프록시처럼 사용해 임의 라우팅이 가능하다.
- **ProxyJump(`-J`)**: 여러 단계의 배스천 호스트를 손쉽게 거쳐 갈 수 있게 해주며, 중첩 SSH 세션을 직접 구성할 필요가 없다.
- **보안 고려사항**: 터널링은 네트워크 통제를 우회할 수 있으므로 서버 측에서 `AllowTcpForwarding`, `PermitOpen`, 사용자별 설정으로 제한할 수 있다.

## Sources

- [[raw/conversations/019e8d48-5843-714e-8928-d757a5983376|019e8d48-5843-714e-8928-d757a5983376]]
