---
id: 019e8db1-ec69-73af-b684-4f78a42af506
title: HTTP vs gRPC vs MQ 비교
topics:
  - http
  - grpc
  - message-queue
  - 마이크로서비스
  - 통신 프로토콜
sources:
  - 019e8db1-624f-73be-ab60-735be949b701
created_at: '2026-06-03T13:35:08.393Z'
updated_at: '2026-06-03T13:35:08.393Z'
---
## Overview

HTTP is mainly used for synchronous communication with external APIs, while [[gRPC]] is suitable for high-performance synchronous communication between internal services. [[MQ]] ([[Kafka]], [[RabbitMQ]], [[SQS]], etc.) is used for asynchronous event delivery and reducing coupling between services. Real large-scale systems use [[HTTP]] + [[gRPC]] + [[MQ]] together.

---

## 한국어

### 개요

[[HTTP]]는 주로 외부 API와의 동기식 통신에 사용되고, [[gRPC]]는 내부 서비스 간 고성능 동기식 통신에 적합하다. [[MQ]]([[Kafka]], [[RabbitMQ]], [[SQS]] 등)는 비동기 이벤트 전달과 서비스 간 결합도 감소에 사용된다. 실제 대규모 시스템은 [[HTTP]] + [[gRPC]] + [[MQ]]를 함께 사용한다.
