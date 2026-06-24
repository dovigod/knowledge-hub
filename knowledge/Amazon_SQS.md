---
id: 019e8db4-07a0-750e-bf50-b01855c89018
name: Amazon SQS
aliases:
  - SQS
  - Simple Queue Service
updated_at: '2026-06-03T13:37:26.432Z'
summary: >-
  Fully managed AWS message queue service for simple, scalable asynchronous
  decoupling between components.
sources:
  - 019e8db1-624f-73be-ab60-735be949b701
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Amazon SQS

## Overview

[[Amazon SQS]] (Simple Queue Service) is AWS's fully managed [[Message Queue]] service. It provides simple, scalable, at-least-once delivery (Standard) or FIFO queues, with no broker to operate.

> [!note] Core traits
> - Fully managed, pay-per-use
> - Standard (high throughput, at-least-once) and FIFO (ordered, exactly-once-ish) queues
> - Native integration with the AWS ecosystem (Lambda, SNS, EventBridge)

## Notes

> [!tip] When to choose SQS
> - You already run on AWS and want zero-ops async decoupling
> - Simple producer→consumer queues without complex routing

> [!example] In a typical stack
> Used as the async layer next to [[HTTP]] (external) and [[gRPC]] (internal sync); compared with [[Kafka]] when streaming/replay isn't needed, or [[RabbitMQ]] when complex routing isn't needed.

> [!warning] Trade-offs
> - Less flexible routing than [[RabbitMQ]]
> - Not a streaming log like [[Kafka]] (no long replay window)
> - AWS-only

---

## 한국어

### 개요

[[Amazon SQS]] (Simple Queue Service)는 AWS의 완전 관리형 [[Message Queue]] 서비스다. Standard(고처리량, at-least-once) 또는 FIFO(순서 보장, exactly-once에 가까운) 큐를 제공하며, 브로커 운영이 필요 없다.

> [!note] 핵심 특성
> - 완전 관리형, 사용량 기반 과금
> - Standard(고처리량, at-least-once)와 FIFO(순서 보장) 큐
> - AWS 생태계(Lambda, SNS, EventBridge)와 네이티브 통합

### 노트

> [!tip] SQS를 선택할 때
> - 이미 AWS 위에서 운영 중이며 운영 부담 없는 비동기 분리가 필요할 때
> - 복잡한 라우팅 없이 단순한 producer→consumer 큐가 필요한 경우

> [!example] 일반적인 스택에서
> [[HTTP]](외부), [[gRPC]](내부 동기)와 함께 비동기 계층으로 사용되며, 스트리밍/재생이 불필요할 때 [[Kafka]]와, 복잡한 라우팅이 불필요할 때 [[RabbitMQ]]와 비교된다.

> [!warning] 트레이드오프
> - [[RabbitMQ]]보다 라우팅 유연성이 낮음
> - [[Kafka]]처럼 긴 재생 윈도우를 가진 스트리밍 로그가 아님
> - AWS 환경에 종속됨

## Sources

- [[raw/conversations/019e8db1-624f-73be-ab60-735be949b701|019e8db1-624f-73be-ab60-735be949b701]]
