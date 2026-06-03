---
id: 019e8cae-9350-7189-83aa-a8dee0c826f5
name: Redis
aliases:
  - Redis
  - redis
  - redis-server
updated_at: '2026-06-03T10:12:01.583Z'
summary: >-
  In-memory key-value data store used as a cache, database, and message broker
  with rich data structures and atomic single-threaded execution.
sources:
  - 019e8cae-3d03-76e8-8213-83715aec185d
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Redis

## Overview

Redis is an in-memory data store, commonly used as a cache, database, and message broker. Data lives in RAM for sub-millisecond reads/writes, with optional persistence to disk via RDB snapshots or AOF logs.

## Notes

- **Data structures**: strings, lists, hashes, sets, sorted sets, streams, bitmaps, HyperLogLog, geospatial, pub/sub.
- **Single-threaded core**: commands execute atomically, enabling simple counters, rate limiters, and distributed locks without explicit locking.
- **Why single-threaded is still fast**: all data lives in RAM (no disk I/O on the hot path), the event loop uses I/O multiplexing (`epoll`/`kqueue`) to handle thousands of connections without thread context switches, and single-threaded execution eliminates lock contention and cache-line bouncing between cores. It's not "just an in-memory map" — the network/event-loop architecture is what makes it scale.
- **Common uses**: caching, session storage, job queues (e.g. BullMQ), rate limiting, leaderboards, real-time pub/sub. Also a typical store for short-lived auth artifacts like [[OIDC]] `state`/`nonce` values keyed with a TTL.
- **When to reach for it**: shared, networked queues/caches across multiple processes or machines.
- **When not to**: single-user local tools where [[SQLite]] is simpler and sufficient — e.g. [[knowledge-hub]] uses SQLite as its Stage 2 extract job queue instead of Redis.

---

## 한국어

### 개요

Redis는 인메모리 데이터 저장소로, 캐시·데이터베이스·메시지 브로커로 흔히 사용된다. 데이터가 RAM에 있어 서브밀리초 단위의 읽기/쓰기가 가능하며, RDB 스냅샷이나 AOF 로그를 통해 디스크에 선택적으로 영속화할 수 있다.

### 노트

- **자료 구조**: 문자열, 리스트, 해시, 셋, 정렬된 셋, 스트림, 비트맵, HyperLogLog, 지리공간, pub/sub.
- **싱글 스레드 코어**: 커맨드가 원자적으로 실행되므로, 명시적 락 없이도 카운터·레이트 리미터·분산 락 같은 패턴을 단순하게 구현할 수 있다.
- **싱글 스레드인데 왜 빠른가**: 모든 데이터가 RAM에 있어 핫 패스에서 디스크 I/O가 없고, 이벤트 루프가 I/O 멀티플렉싱(`epoll`/`kqueue`)을 사용해 스레드 컨텍스트 스위치 없이 수천 개의 연결을 처리하며, 싱글 스레드 실행 덕분에 락 경합과 코어 간 캐시 라인 바운싱이 사라진다. "그냥 인메모리 맵"이 아니라 네트워크/이벤트 루프 아키텍처가 성능의 핵심이다.
- **흔한 용도**: 캐싱, 세션 저장소, 작업 큐(BullMQ 등), 레이트 리미팅, 리더보드, 실시간 pub/sub. [[OIDC]]의 `state`/`nonce`처럼 TTL을 걸어 짧게 유지하는 인증 아티팩트 저장소로도 자주 쓰인다.
- **언제 쓰는가**: 여러 프로세스나 머신에서 공유해야 하는, 네트워크 너머의 큐/캐시가 필요할 때.
- **언제 쓰지 않는가**: [[SQLite]]로 충분히 단순하게 풀리는 단일 사용자 로컬 도구. 예를 들어 [[knowledge-hub]]는 Stage 2 extract 잡 큐를 Redis 대신 SQLite로 처리한다.

## Sources

- [[raw/conversations/019e8cae-3d03-76e8-8213-83715aec185d|019e8cae-3d03-76e8-8213-83715aec185d]]
- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
