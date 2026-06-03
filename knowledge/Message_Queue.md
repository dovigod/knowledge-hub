---
id: 019e8db3-f969-7419-96af-a1ed1b5c5280
name: Message Queue
aliases:
  - MQ
  - event bus
  - message broker
  - 메시지 큐
updated_at: '2026-06-03T13:37:22.793Z'
summary: >-
  Asynchronous messaging middleware that decouples producers and consumers via
  durable queues or event streams.
sources:
  - 019e8db1-624f-73be-ab60-735be949b701
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Message Queue

## Overview

A [[Message Queue]] (MQ) is asynchronous messaging middleware that sits between producers and consumers, buffering events so they can be processed independently. It is the foundation of event-driven architectures and is used to **reduce coupling, absorb load spikes, and enable async workflows** between services.

> [!note] Common implementations
> - [[Kafka]] — high-throughput distributed log/event stream
> - [[RabbitMQ]] — flexible broker with rich routing semantics
> - [[Amazon SQS]] — managed queue on AWS

## Notes

> [!tip] When to choose MQ
> - Producer and consumer should not block on each other
> - Workloads need backpressure or retry tolerance
> - Multiple consumers want to react to the same event (pub/sub)

> [!example] System composition
> Large systems pair [[HTTP]] (external sync), [[gRPC]] (internal sync), and a [[Message Queue]] (async/decoupled events) to get the strengths of each.

> [!warning] Trade-offs
> - Eventual consistency, not request/response
> - Operational complexity (broker ops, ordering, exactly-once semantics)
> - Harder to debug end-to-end flows than synchronous calls

---

## 한국어

### 개요

[[Message Queue]](MQ)는 생산자(producer)와 소비자(consumer) 사이에 위치하는 비동기 메시징 미들웨어로, 이벤트를 버퍼링하여 양쪽이 독립적으로 처리할 수 있게 한다. 이벤트 기반 아키텍처의 핵심이며, **서비스 간 결합도 감소, 트래픽 스파이크 흡수, 비동기 워크플로우 구현**에 사용된다.

> [!note] 대표 구현체
> - [[Kafka]] — 고처리량 분산 로그/이벤트 스트림
> - [[RabbitMQ]] — 풍부한 라우팅을 지원하는 유연한 브로커
> - [[Amazon SQS]] — AWS의 관리형 큐 서비스

### 노트

> [!tip] MQ를 선택할 때
> - 생산자와 소비자가 서로 블로킹되지 않아야 할 때
> - 백프레셔 또는 재시도 내성이 필요한 워크로드
> - 동일 이벤트에 여러 소비자가 반응해야 할 때(pub/sub)

> [!example] 시스템 구성
> 대규모 시스템은 [[HTTP]](외부 동기), [[gRPC]](내부 동기), [[Message Queue]](비동기·결합도 감소 이벤트)를 함께 사용해 각 방식의 장점을 취한다.

> [!warning] 트레이드오프
> - 요청/응답이 아닌 결과적 일관성(eventual consistency)
> - 브로커 운영, 순서 보장, exactly-once 등 운영 복잡도
> - 동기 호출 대비 종단 간 흐름 디버깅이 어려움

## Sources

- [[raw/conversations/019e8db1-624f-73be-ab60-735be949b701|019e8db1-624f-73be-ab60-735be949b701]]
