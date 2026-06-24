---
id: 019e8db3-f972-767e-b820-ca75527aafb8
name: RabbitMQ
aliases:
  - rabbit
  - rabbitmq
updated_at: '2026-06-03T13:37:22.802Z'
summary: >-
  Traditional message broker offering flexible routing (exchanges, queues,
  bindings) for async service communication.
sources:
  - 019e8db1-624f-73be-ab60-735be949b701
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# RabbitMQ

## Overview

[[RabbitMQ]] is a mature open-source message broker built around AMQP, offering rich routing through exchanges, queues, and bindings. It is a common choice for general-purpose [[Message Queue]] needs where flexible delivery semantics matter more than raw throughput.

> [!note] Core traits
> - Exchange → queue routing (direct, topic, fanout, headers)
> - Per-message acks, DLQ, priorities
> - Strong tooling and management UI

## Notes

> [!tip] When to choose RabbitMQ
> - Work queues, RPC-over-MQ, fan-out with complex routing rules
> - Mid-throughput workloads needing reliable delivery semantics

> [!example] In a typical stack
> Acts as the async layer alongside [[HTTP]] (external) and [[gRPC]] (internal sync); compared to [[Kafka]] when streaming/replay is less important.

> [!warning] Trade-offs
> - Lower raw throughput than [[Kafka]]
> - Not ideal for long-term event log retention

---

## 한국어

### 개요

[[RabbitMQ]]는 AMQP 기반의 성숙한 오픈소스 메시지 브로커로, exchange·queue·binding을 통한 풍부한 라우팅을 제공한다. 순수 처리량보다 유연한 전달 의미가 중요한 일반적 [[Message Queue]] 요구사항에 많이 쓰인다.

> [!note] 핵심 특성
> - exchange → queue 라우팅 (direct, topic, fanout, headers)
> - 메시지별 ack, DLQ, 우선순위 지원
> - 강력한 도구와 관리 UI

### 노트

> [!tip] RabbitMQ를 선택할 때
> - 작업 큐, MQ 기반 RPC, 복잡한 라우팅 규칙을 가진 fan-out
> - 신뢰성 있는 전달 의미가 필요한 중간 처리량 워크로드

> [!example] 일반적인 스택에서
> [[HTTP]](외부), [[gRPC]](내부 동기)와 함께 비동기 계층으로 사용되며, 스트리밍/재생이 덜 중요할 때 [[Kafka]]와 비교된다.

> [!warning] 트레이드오프
> - [[Kafka]] 대비 순수 처리량이 낮음
> - 장기 이벤트 로그 보존 용도에는 부적합

## Sources

- [[raw/conversations/019e8db1-624f-73be-ab60-735be949b701|019e8db1-624f-73be-ab60-735be949b701]]
