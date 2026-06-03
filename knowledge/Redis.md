---
id: 019e8cae-9350-7189-83aa-a8dee0c826f5
name: Redis
aliases:
  - redis
  - redis-server
updated_at: '2026-06-03T08:51:51.760Z'
summary: >-
  In-memory key-value data store used as a cache, database, and message broker
  with rich data structures and atomic single-threaded execution.
sources:
  - 019e8cae-3d03-76e8-8213-83715aec185d
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Redis

## Overview

Redis is an in-memory data store, commonly used as a cache, database, and message broker. Data lives in RAM for sub-millisecond reads/writes, with optional persistence to disk via RDB snapshots or AOF logs.

## Notes

- **Data structures**: strings, lists, hashes, sets, sorted sets, streams, bitmaps, HyperLogLog, geospatial, pub/sub.
- **Single-threaded core**: commands execute atomically, enabling simple counters, rate limiters, and distributed locks without explicit locking.
- **Common uses**: caching, session storage, job queues (e.g. BullMQ), rate limiting, leaderboards, real-time pub/sub.
- **When to reach for it**: shared, networked queues/caches across multiple processes or machines.
- **When not to**: single-user local tools where [[SQLite]] is simpler and sufficient — e.g. [[knowledge-hub]] uses SQLite as its Stage 2 extract job queue instead of Redis.

## Sources

- [[raw/conversations/019e8cae-3d03-76e8-8213-83715aec185d|019e8cae-3d03-76e8-8213-83715aec185d]]
