---
id: 019e8db1-72c6-702a-a0a8-c5e64be266ae
name: Queries Per Second
aliases:
  - QPS
  - qps
  - queries per second
updated_at: '2026-06-03T13:34:37.254Z'
summary: >-
  Throughput metric measuring the number of requests a system processes per
  second.
sources:
  - 019e8db1-0369-703e-8553-fedfe19e0cee
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Queries Per Second

## Overview

Queries Per Second (QPS) is a throughput metric that counts how many requests a system processes in one second. It applies to HTTP requests, RPC calls, database queries, and any request-driven workload.

> [!note] Definition
> QPS = number of completed requests / elapsed seconds. Higher QPS means more load on the system.

## Notes

- Commonly used alongside [[Latency]] and [[Error Rate]] — high QPS alone is meaningless without knowing response time and failure ratio.
- Applies to many request types: HTTP endpoints, RPC services, [[Database]] queries, cache lookups.
- Related throughput metrics: [[RPS]] (Requests Per Second), [[TPS]] (Transactions Per Second).
- Used in capacity planning, load testing, and SLO/SLA definitions.

> [!tip] Reading QPS
> Always pair QPS with p50/p95/p99 [[Latency]] and [[Error Rate]] to assess true system health. A system can sustain high QPS while silently degrading.

> [!warning] Common pitfall
> Peak QPS ≠ sustainable QPS. Benchmarks often report burst numbers that cannot be held under steady load.

---

## 한국어

### 개요

QPS(Queries Per Second)는 시스템이 1초 동안 처리한 요청 수를 나타내는 처리량(throughput) 지표다. HTTP 요청, RPC 호출, 데이터베이스 쿼리 등 요청 기반 워크로드 전반에 적용된다.

> [!note] 정의
> QPS = 완료된 요청 수 / 경과 시간(초). QPS가 높을수록 시스템 부하가 크다.

### 노트

- [[Latency]]와 [[Error Rate]]를 함께 봐야 의미가 있다 — 응답 시간과 실패율을 모르면 QPS 수치만으로는 시스템 상태를 판단할 수 없다.
- 다양한 요청 유형에 적용된다: HTTP 엔드포인트, RPC 서비스, [[Database]] 쿼리, 캐시 조회 등.
- 관련 처리량 지표: [[RPS]](Requests Per Second), [[TPS]](Transactions Per Second).
- 용량 산정(capacity planning), 부하 테스트, SLO/SLA 정의 등에 사용된다.

> [!tip] QPS 읽는 법
> QPS는 항상 p50/p95/p99 [[Latency]] 및 [[Error Rate]]와 함께 봐야 실제 시스템 건강도를 알 수 있다. 높은 QPS를 유지하면서도 내부적으로는 성능이 저하될 수 있다.

> [!warning] 흔한 함정
> 피크 QPS ≠ 지속 가능한 QPS. 벤치마크 수치는 보통 짧은 순간의 버스트 값이며, 정상 부하에서 장시간 유지되지 않을 수 있다.

## Sources

- [[raw/conversations/019e8db1-0369-703e-8553-fedfe19e0cee|019e8db1-0369-703e-8553-fedfe19e0cee]]
