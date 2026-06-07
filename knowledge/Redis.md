---
id: 019e8cae-9350-7189-83aa-a8dee0c826f5
name: Redis
aliases:
  - Redis
  - redis
  - redis-server
updated_at: '2026-06-07T08:28:17.053Z'
summary: >-
  In-memory key-value data store used as a cache, database, and message broker
  with rich data structures and atomic single-threaded execution.
sources:
  - 019e8cae-3d03-76e8-8213-83715aec185d
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
  - 019e8cf7-f7e2-72c3-8125-d8541b33763c
  - 019e8daf-09b5-7403-af7c-3149b422f8c2
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Redis

## Overview

Redis is an in-memory data store, commonly used as a cache, database, and message broker. Data lives in RAM for sub-millisecond reads/writes, with optional persistence to disk via RDB snapshots or AOF logs.

> [!note] Single-threaded but fast
> The network/event-loop architecture (I/O multiplexing + no lock contention) is what makes Redis scale — not just "in-memory map."

## Notes

- **Data structures**: strings, lists, hashes, sets, sorted sets, streams, bitmaps, HyperLogLog, geospatial, pub/sub.
- **Single-threaded core**: commands execute atomically, enabling simple counters, rate limiters, and distributed locks without explicit locking.
- **Why single-threaded is still fast**: all data lives in RAM (no disk I/O on the hot path), the event loop uses I/O multiplexing (`epoll`/`kqueue`) to handle thousands of connections without thread context switches, and single-threaded execution eliminates lock contention and cache-line bouncing between cores.
- **In-memory ≠ just a map**: a plain in-process `Map` is faster per-op but lives in one process. Redis adds networked sharing across processes/machines, atomic command semantics, native data structures (sorted sets, streams, …), TTLs, and optional persistence — that's what justifies the network hop.
- **Common uses**: caching, session storage, job queues (e.g. BullMQ), rate limiting, leaderboards, real-time pub/sub. Also a typical store for short-lived [[OIDC]] / [[OAuth 2.0]] auth artifacts like `state`/`nonce` values keyed with a TTL.
- **When to reach for it**: shared, networked queues/caches across multiple processes or machines.
- **When not to**: single-user local tools where [[SQLite]] is simpler and sufficient — e.g. [[knowledge-hub]] uses SQLite as its Stage 2 extract job queue instead of Redis.

## Connection URL schemes

Redis connections use two URL schemes that signal whether transport-layer [[TLS]] is enforced:

- `redis://` → plaintext TCP (default port `6379`).
- `rediss://` → TLS-encrypted connection (typically port `6380` or a configured TLS port). The extra `s` is an explicit signal that TLS is required, not optional.

General form:

```
rediss://[user[:password]@]host:port/db
```

Example:

```
rediss://user:password@redis.example.com:6380/0
```

> [!tip] Managed Redis defaults to `rediss://`
> Cloud-managed Redis (AWS ElastiCache with in-transit encryption, Upstash, Redis Cloud, etc.) almost always requires `rediss://`. Treat plain `redis://` as for local/dev only.

> [!warning] Client-side gotchas
> - **Client support**: TLS must be enabled explicitly in some clients (e.g. `ioredis`, `node-redis`) — the scheme alone isn't always enough.
> - **Certificate validation**: self-signed or internal CA setups may need `rejectUnauthorized: false` or a custom CA bundle.
> - **Performance**: TLS adds handshake + per-message encryption overhead, but this is usually negligible vs. network latency.

## Examples

```ts
// node-redis with rediss:// — TLS inferred from scheme
import { createClient } from 'redis'
const client = createClient({ url: 'rediss://user:pw@redis.example.com:6380/0' })
await client.connect()
```

---

## 한국어

### 개요

Redis는 인메모리 데이터 저장소로, 캐시·데이터베이스·메시지 브로커로 흔히 사용된다. 데이터가 RAM에 있어 서브밀리초 단위의 읽기/쓰기가 가능하며, RDB 스냅샷이나 AOF 로그를 통해 디스크에 선택적으로 영속화할 수 있다.

> [!note] 싱글 스레드인데 빠른 이유
> 네트워크/이벤트 루프 아키텍처(I/O 멀티플렉싱 + 락 경합 없음)가 Redis 성능의 핵심이다. "그냥 인메모리 맵"이 아니다.

### 노트

- **자료 구조**: 문자열, 리스트, 해시, 셋, 정렬된 셋, 스트림, 비트맵, HyperLogLog, 지리공간, pub/sub.
- **싱글 스레드 코어**: 커맨드가 원자적으로 실행되므로, 명시적 락 없이도 카운터·레이트 리미터·분산 락 같은 패턴을 단순하게 구현할 수 있다.
- **싱글 스레드인데 왜 빠른가**: 모든 데이터가 RAM에 있어 핫 패스에서 디스크 I/O가 없고, 이벤트 루프가 I/O 멀티플렉싱(`epoll`/`kqueue`)을 사용해 스레드 컨텍스트 스위치 없이 수천 개의 연결을 처리하며, 싱글 스레드 실행 덕분에 락 경합과 코어 간 캐시 라인 바운싱이 사라진다.
- **인메모리 ≠ 그냥 맵**: 프로세스 내부의 `Map`은 단일 연산 속도는 더 빠르지만 한 프로세스에 갇혀 있다. Redis는 프로세스/머신 간 네트워크 공유, 커맨드 원자성, 내장 자료구조(정렬된 셋, 스트림 등), TTL, 선택적 영속화를 더해 주며, 이게 네트워크 홉을 정당화한다.
- **흔한 용도**: 캐싱, 세션 저장소, 작업 큐(BullMQ 등), 레이트 리미팅, 리더보드, 실시간 pub/sub. [[OIDC]] / [[OAuth 2.0]]의 `state`/`nonce`처럼 TTL을 걸어 짧게 유지하는 인증 아티팩트 저장소로도 자주 쓰인다.
- **언제 쓰는가**: 여러 프로세스나 머신에서 공유해야 하는, 네트워크 너머의 큐/캐시가 필요할 때.
- **언제 쓰지 않는가**: [[SQLite]]로 충분히 단순하게 풀리는 단일 사용자 로컬 도구. 예를 들어 [[knowledge-hub]]는 Stage 2 extract 잡 큐를 Redis 대신 SQLite로 처리한다.

### 연결 URL 스킴

Redis 연결 URL은 전송 계층에서 [[TLS]]를 강제하는지 여부를 두 가지 스킴으로 구분한다:

- `redis://` → 평문 TCP 연결 (기본 포트 `6379`).
- `rediss://` → TLS로 암호화된 연결 (보통 포트 `6380` 또는 설정된 TLS 포트). 끝의 `s`는 "TLS를 명시적으로 강제한다"는 신호이지 선택적인 것이 아니다.

일반 형식:

```
rediss://[user[:password]@]host:port/db
```

예시:

```
rediss://user:password@redis.example.com:6380/0
```

> [!tip] 매니지드 Redis는 사실상 `rediss://`가 기본
> 클라우드 매니지드 Redis(AWS ElastiCache의 in-transit encryption, Upstash, Redis Cloud 등)는 거의 항상 `rediss://`를 요구한다. 평문 `redis://`는 로컬/개발 환경 용도로만 보는 게 안전하다.

> [!warning] 클라이언트 쪽 주의사항
> - **클라이언트 지원**: 일부 클라이언트(`ioredis`, `node-redis` 등)는 스킴만으로는 부족하고 TLS 옵션을 별도로 켜야 한다.
> - **인증서 검증**: self-signed 또는 사내 CA 환경에서는 `rejectUnauthorized: false`나 별도 CA 번들 설정이 필요할 수 있다.
> - **성능**: TLS handshake와 메시지별 암호화 오버헤드가 있지만, 네트워크 지연 대비 영향은 일반적으로 크지 않다.

### 예시

```ts
// node-redis에서 rediss:// 사용 — 스킴만으로 TLS가 적용된다
import { createClient } from 'redis'
const client = createClient({ url: 'rediss://user:pw@redis.example.com:6380/0' })
await client.connect()
```

## Sources

- [[raw/conversations/019e8cae-3d03-76e8-8213-83715aec185d|019e8cae-3d03-76e8-8213-83715aec185d]]
- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
- [[raw/conversations/019e8cf7-f7e2-72c3-8125-d8541b33763c|019e8cf7-f7e2-72c3-8125-d8541b33763c]]
- [[raw/conversations/019e8daf-09b5-7403-af7c-3149b422f8c2|019e8daf-09b5-7403-af7c-3149b422f8c2]]
