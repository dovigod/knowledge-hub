---
id: 019e8db3-f970-746b-838d-d4cbc2383b11
name: Kafka
aliases:
  - Apache Kafka
  - kafka
updated_at: '2026-06-03T13:37:22.800Z'
summary: >-
  Distributed event streaming platform used as a high-throughput, durable log
  for async messaging and event sourcing.
sources:
  - 019e8db1-624f-73be-ab60-735be949b701
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Kafka

## Overview

[[Kafka]] is a distributed event streaming platform that stores messages as an append-only log partitioned across brokers. It is a common backbone for high-throughput [[Message Queue]] use cases, event sourcing, and stream processing.

> [!note] Core traits
> - Append-only, partitioned, replicated log
> - Consumer groups with offset-based replay
> - Designed for very high throughput and durable retention

## Notes

> [!tip] When to choose Kafka
> - High-volume event pipelines (logs, metrics, clickstreams)
> - Replayable history is valuable (audit, reprocessing)
> - Multiple independent consumers of the same stream

> [!example] In a typical stack
> Used as the async backbone alongside [[HTTP]] (external) and [[gRPC]] (internal sync).

> [!warning] Trade-offs
> - Operationally heavier than [[RabbitMQ]] or [[Amazon SQS]]
> - Not optimized for complex per-message routing

---

## 한국어

### 개요

[[Kafka]]는 메시지를 브로커에 파티션 단위로 분산 저장하는 append-only 로그 기반의 분산 이벤트 스트리밍 플랫폼이다. 고처리량 [[Message Queue]] 시나리오, 이벤트 소싱, 스트림 처리의 백본으로 널리 쓰인다.

> [!note] 핵심 특성
> - append-only, 파티셔닝, 복제되는 로그 구조
> - 컨슈머 그룹 + 오프셋 기반 재처리
> - 매우 높은 처리량과 내구성 있는 보존 기간 설계

### 노트

> [!tip] Kafka를 선택할 때
> - 대용량 이벤트 파이프라인(로그, 메트릭, 클릭스트림)
> - 이력 재생(audit, 재처리)이 가치 있을 때
> - 동일 스트림을 독립적으로 소비하는 여러 컨슈머

> [!example] 일반적인 스택에서
> [[HTTP]](외부), [[gRPC]](내부 동기)와 함께 비동기 백본으로 사용된다.

> [!warning] 트레이드오프
> - [[RabbitMQ]], [[Amazon SQS]]보다 운영 부담이 큼
> - 메시지별 복잡한 라우팅에는 최적화되어 있지 않음

## Sources

- [[raw/conversations/019e8db1-624f-73be-ab60-735be949b701|019e8db1-624f-73be-ab60-735be949b701]]
