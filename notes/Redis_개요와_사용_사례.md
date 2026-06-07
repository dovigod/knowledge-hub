---
id: 019ea115-c6f9-727f-9a3e-8931db84bce4
title: Redis 개요와 사용 사례
topics:
  - redis
  - in-memory
  - cache
  - 데이터 저장소
sources:
  - 019e8cae-3d03-76e8-8213-83715aec185d
created_at: '2026-06-07T07:56:59.513Z'
updated_at: '2026-06-07T07:56:59.513Z'
---
## Overview

Redis is an in-memory data store, commonly used as a cache, database, and message broker.

- In-memory: data lives in RAM, sub-millisecond reads/writes; persistence to disk optional ([[RDB]] snapshots, [[AOF]] logs).
- Key-value with rich data structures: strings, lists, hashes, sets, sorted sets, streams, bitmaps, HyperLogLog, geospatial, pub/sub.
- Single-threaded core: commands execute atomically, enabling simple counters, rate limiters, distributed locks.
- Common uses: caching, session storage, job queues (e.g. [[BullMQ]]), rate limiting, leaderboards, real-time pub/sub.

## Repo-relevant note

[[knowledge-hub]] deliberately avoids [[Redis]] — it uses [[SQLite]] as the job queue for Stage 2 extract jobs, the right call for a local single-user tool. [[Redis]] makes sense for shared, networked queues/caches across multiple processes or machines.

---

## 한국어

### 개요

[[Redis]]는 인메모리 데이터 저장소로, 일반적으로 캐시, 데이터베이스, 메시지 브로커로 사용된다.

- 인메모리: 데이터가 RAM에 존재하며, 밀리초 미만의 읽기/쓰기 성능; 디스크 영속성은 선택 사항 ([[RDB]] 스냅샷, [[AOF]] 로그).
- 풍부한 데이터 구조를 가진 키-값 저장소: 문자열, 리스트, 해시, 셋, 정렬된 셋, 스트림, 비트맵, HyperLogLog, 지리공간, pub/sub.
- 단일 스레드 코어: 명령어가 원자적으로 실행되어, 단순한 카운터, 레이트 리미터, 분산 락 구현이 용이.
- 일반적인 용도: 캐싱, 세션 저장소, 작업 큐 (예: [[BullMQ]]), 레이트 리미팅, 리더보드, 실시간 pub/sub.

### 레포 관련 참고

[[knowledge-hub]]는 의도적으로 [[Redis]]를 사용하지 않는다 — Stage 2 extract 작업의 작업 큐로 [[SQLite]]를 사용하며, 이는 로컬 단일 사용자 도구에 적합한 선택이다. [[Redis]]는 여러 프로세스나 머신에 걸쳐 공유되는 네트워크 큐/캐시에 적합하다.
