---
id: 019e8db1-0eef-7468-a453-13ceab1745d4
name: ioredis
aliases:
  - ioredis client
updated_at: '2026-06-03T13:34:11.695Z'
summary: 'Popular Node.js Redis client with cluster, sentinel, and TLS support.'
sources:
  - 019e8daf-09b5-7403-af7c-3149b422f8c2
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# ioredis

## Overview

[[ioredis]] is a widely used [[Node.js]] client for [[Redis]]. It supports standalone, Sentinel, and Cluster deployments, pipelining, transactions, pub/sub, and Lua scripting, and has first-class [[TLS]] support for `rediss://` connections.

> [!tip] TLS connections
> When connecting with `rediss://`, pass a `tls` option (even an empty `{}`) so ioredis upgrades the socket to TLS.

## Notes

- Accepts either a URL (`redis://` / `rediss://`) or an options object.
- For self-signed certificates, set `tls.rejectUnauthorized = false` or provide a custom CA in `tls.ca`.
- Common alternative: `node-redis` (the official client), which exposes TLS via `socket.tls`.

> [!example] Connecting over TLS
> ```ts
> import Redis from 'ioredis'
> const client = new Redis('rediss://user:pass@host:6380/0', { tls: {} })
> ```

---

## 한국어

### 개요

[[ioredis]]는 널리 쓰이는 [[Node.js]]용 [[Redis]] 클라이언트다. 단일 노드, Sentinel, Cluster 배포를 모두 지원하며 파이프라이닝, 트랜잭션, pub/sub, Lua 스크립팅을 제공하고 `rediss://` 연결을 위한 [[TLS]] 지원도 기본 내장돼 있다.

> [!tip] TLS 연결
> `rediss://`로 연결할 때는 `tls` 옵션(빈 `{}`라도)을 넘겨야 ioredis가 소켓을 TLS로 업그레이드한다.

### 노트

- URL(`redis://` / `rediss://`) 또는 옵션 객체 모두 허용한다.
- self-signed 인증서를 쓰는 경우 `tls.rejectUnauthorized = false`로 두거나 `tls.ca`에 커스텀 CA를 넣는다.
- 대표적 대안은 공식 클라이언트인 `node-redis`이며, TLS 설정은 `socket.tls`로 노출된다.

> [!example] TLS 연결 예
> ```ts
> import Redis from 'ioredis'
> const client = new Redis('rediss://user:pass@host:6380/0', { tls: {} })
> ```

## Sources

- [[raw/conversations/019e8daf-09b5-7403-af7c-3149b422f8c2|019e8daf-09b5-7403-af7c-3149b422f8c2]]
