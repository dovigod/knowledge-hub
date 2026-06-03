---
id: 019e8d4a-33eb-726c-81a8-c4346946c1a9
name: known_hosts
aliases:
  - ssh known_hosts
  - ~/.ssh/known_hosts
updated_at: '2026-06-03T11:41:50.955Z'
summary: >-
  The SSH client file that stores trusted server host public keys so the client
  can detect spoofed or changed servers on subsequent connections.
sources:
  - 019e8d48-5843-714e-8928-d757a5983376
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# known_hosts

## Overview
`known_hosts` is the SSH client-side file (typically `~/.ssh/known_hosts`) that records the public host keys of servers the user has previously connected to. On each connection, SSH compares the server's presented host key against this file to detect impersonation or man-in-the-middle attacks.

## Notes
- **TOFU model**: on the first connection, SSH prompts the user to accept the host key and then stores it. Subsequent connections silently verify against the stored entry.
- **Mismatch behavior**: if the server's key changes (legitimate reinstall *or* an attack), SSH refuses to connect and prints a `REMOTE HOST IDENTIFICATION HAS CHANGED` warning.
- **Management**: entries can be removed with `ssh-keygen -R hostname` after a legitimate server rebuild.
- **Hashing**: entries may be hashed (`HashKnownHosts yes`) to prevent leaking the list of hosts a user has accessed.
- **System-wide variant**: `/etc/ssh/ssh_known_hosts` provides a shared trusted list for all users on a machine.

---

## 한국어

### 개요
`known_hosts`는 SSH 클라이언트가 이전에 접속했던 서버의 호스트 공개키를 저장하는 파일이다(보통 `~/.ssh/known_hosts`). 새 접속이 일어날 때마다 서버가 제시한 호스트 키를 이 파일과 비교해 위장이나 중간자 공격을 감지한다.

### 노트
- **TOFU 모델**: 첫 접속에서 사용자가 호스트 키를 수락하면 파일에 저장된다. 이후 접속은 저장된 키와 자동으로 비교된다.
- **불일치 동작**: 서버의 키가 바뀌면(정상적인 재설치든 공격이든) SSH는 접속을 거부하고 `REMOTE HOST IDENTIFICATION HAS CHANGED` 경고를 띄운다.
- **관리**: 서버를 정상적으로 재구성한 뒤에는 `ssh-keygen -R hostname` 명령으로 항목을 제거할 수 있다.
- **해싱 옵션**: `HashKnownHosts yes` 설정을 사용하면 접속 이력이 노출되지 않도록 호스트명을 해시 형태로 저장한다.
- **시스템 공용 파일**: `/etc/ssh/ssh_known_hosts`는 머신의 모든 사용자가 공유하는 신뢰 호스트 목록이다.

## Sources

- [[raw/conversations/019e8d48-5843-714e-8928-d757a5983376|019e8d48-5843-714e-8928-d757a5983376]]
