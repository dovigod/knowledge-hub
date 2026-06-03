---
id: 019e8cf7-f7b2-760a-93dd-0613de845b27
name: I/O Multiplexing
aliases:
  - epoll
  - event loop
  - io-multiplexing
  - kqueue
  - select-poll-epoll
updated_at: '2026-06-03T10:12:01.586Z'
summary: >-
  A kernel-assisted mechanism (select/poll/epoll/kqueue) that lets a single
  thread monitor many file descriptors and act only on those ready for I/O.
sources:
  - 019e8cf5-a947-70fe-8a72-b2a2fcda81aa
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# I/O Multiplexing

## Overview
I/O multiplexing lets one thread wait on many sockets (or file descriptors) simultaneously and wake up only when one of them is ready to read or write. It is the kernel primitive that makes single-threaded event-loop servers — Node.js, nginx, Redis — handle tens of thousands of concurrent connections.

## Notes
- POSIX APIs: `select` (O(n), fd limit), `poll` (O(n), no limit). Linux: `epoll` (O(1), edge/level-triggered). BSD/macOS: `kqueue`. Modern Linux: `io_uring` for full async I/O.
- Compared to thread-per-connection: avoids context switches, kernel-thread overhead, and lock contention; one CPU core can saturate a NIC.
- Trade-off: any CPU-heavy work in the event loop blocks every other connection — hence Redis avoids `O(n)` commands, Node.js offloads CPU work to worker threads.
- Pairs naturally with non-blocking sockets — `read()`/`write()` return immediately with `EAGAIN` instead of sleeping.

---

## 한국어

### 개요
I/O 다중화는 단일 스레드가 다수의 소켓(또는 file descriptor)을 동시에 대기하다가 read/write 준비가 된 것만 깨우게 해줍니다. Node.js, nginx, Redis 같은 단일 스레드 event loop 서버가 수만 개의 동시 연결을 처리할 수 있게 하는 커널 프리미티브입니다.

### 노트
- POSIX API: `select` (O(n), fd 제한), `poll` (O(n), 제한 없음). Linux: `epoll` (O(1), edge/level-triggered). BSD/macOS: `kqueue`. 최신 Linux: 완전 비동기 I/O를 위한 `io_uring`.
- thread-per-connection 모델과 비교: 컨텍스트 스위치, 커널 스레드 오버헤드, 락 경합을 피하며 단일 CPU 코어로 NIC을 포화시킬 수 있습니다.
- 트레이드오프: event loop에서 CPU 집약적인 작업은 다른 모든 연결을 블록합니다 — 그래서 Redis는 `O(n)` 명령을 피하고, Node.js는 CPU 작업을 worker thread로 offload합니다.
- non-blocking 소켓과 자연스럽게 짝을 이룹니다 — `read()`/`write()`가 sleep 대신 `EAGAIN`을 즉시 반환합니다.

## Sources

- [[raw/conversations/019e8cf5-a947-70fe-8a72-b2a2fcda81aa|019e8cf5-a947-70fe-8a72-b2a2fcda81aa]]
