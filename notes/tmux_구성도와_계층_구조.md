---
id: 019e9be3-6682-769b-b374-6a418cb1b6d7
title: tmux 구성도와 계층 구조
topics:
  - tmux
  - 터미널 멀티플렉서
  - 세션 관리
sources:
  - 019e9be2-e8f3-7388-975a-2600c07b1b31
created_at: '2026-06-06T07:43:51.938Z'
updated_at: '2026-06-06T07:43:51.938Z'
---
## Overview

[[tmux]] is a terminal multiplexer that allows you to manage multiple terminals within a single SSH session or terminal.

Once you understand the structure, most commands become naturally understandable.

## Overall Structure

```text
tmux Server
│
├── Session
│   ├── Window
│   │   ├── Pane
│   │   └── Pane
│   │
│   └── Window
│       ├── Pane
│       └── Pane
│
└── Session
    └── Window
        └── Pane
```

### 1. Server

When you first run tmux, a tmux server process is created in the background.

```bash
tmux
```

In reality:

```text
tmux server
└── session
```

is created.

Check:

```bash
ps aux | grep tmux
```

---

### 2. Session

The top-level unit of work.

For example:

```bash
tmux new -s backend
tmux new -s frontend
tmux new -s monitoring
```

Structure:

```text
tmux server
├── backend
├── frontend
└── monitoring
```

A session is an independent workspace.

Examples:

- backend → API server
- frontend → React development
- monitoring → log monitoring

Even if [[SSH]] disconnects, the session stays alive.

List:

```bash
tmux ls
```

---

### 3. Window

A tab concept inside a session.

```text
Session: backend

Window 0: editor
Window 1: logs
Window 2: shell
```

Similar to browser tabs.

Create:

```bash
Ctrl+b c
```

Navigate:

```bash
Ctrl+b n
Ctrl+b p
```

---

### 4. Pane

The actual terminal that splits a window.

Example:

```text
+------------------+
| editor           |
+---------+--------+
| build   | logs   |
+---------+--------+
```

Create:

Horizontal split:

```bash
Ctrl+b "
```

Vertical split:

```bash
Ctrl+b %
```

Navigate:

```bash
Ctrl+b arrow-keys
```

---

## Attach / Detach Concepts

### Attach

```bash
tmux attach -t backend
```

### Detach

```bash
Ctrl+b d
```

The session is preserved and only the terminal is detached.

---

## Hierarchy to Remember at the Core

```text
tmux Server
    ↓
Session
    ↓
Window
    ↓
Pane
```

In practice, you typically configure it as **project unit = Session**, **role unit = Window**, **individual running terminal = Pane**.

---

## 한국어

[[tmux]]는 터미널 멀티플렉서(Terminal Multiplexer)로, 하나의 SSH 세션이나 터미널 안에서 여러 개의 터미널을 관리할 수 있게 해준다.

구조를 이해하면 대부분의 명령어가 자연스럽게 이해된다.

### 전체 구조

```text
tmux Server
│
├── Session (세션)
│   ├── Window (윈도우)
│   │   ├── Pane (패널)
│   │   └── Pane
│   │
│   └── Window
│       ├── Pane
│       └── Pane
│
└── Session
    └── Window
        └── Pane
```

### 1. Server

tmux를 처음 실행하면 백그라운드에 tmux 서버 프로세스가 생성된다.

```bash
tmux
```

실제로는:

```text
tmux server
└── session
```

가 만들어진다.

확인:

```bash
ps aux | grep tmux
```

---

### 2. Session

가장 상위 작업 단위.

예를 들어:

```bash
tmux new -s backend
tmux new -s frontend
tmux new -s monitoring
```

구조:

```text
tmux server
├── backend
├── frontend
└── monitoring
```

세션은 독립적인 작업 공간이다.

예:

- backend → API 서버
- frontend → React 개발
- monitoring → 로그 모니터링

[[SSH]]가 끊겨도 세션은 살아있다.

목록 확인:

```bash
tmux ls
```

---

### 3. Window

세션 안의 탭(Tab) 개념.

```text
Session: backend

Window 0: editor
Window 1: logs
Window 2: shell
```

브라우저 탭과 비슷하다.

생성:

```bash
Ctrl+b c
```

이동:

```bash
Ctrl+b n
Ctrl+b p
```

---

### 4. Pane

윈도우를 분할한 실제 터미널.

예:

```text
+------------------+
| editor           |
+---------+--------+
| build   | logs   |
+---------+--------+
```

생성:

수평 분할:

```bash
Ctrl+b "
```

수직 분할:

```bash
Ctrl+b %
```

이동:

```bash
Ctrl+b 방향키
```

---

### Attach / Detach 개념

#### Attach

```bash
tmux attach -t backend
```

#### Detach

```bash
Ctrl+b d
```

세션은 유지되고 터미널만 분리된다.

---

### 핵심적으로 기억할 계층

```text
tmux Server
    ↓
Session
    ↓
Window
    ↓
Pane
```

실무에서는 보통 **프로젝트 단위 = Session**, **역할 단위 = Window**, **실행 중인 개별 터미널 = Pane** 으로 구성한다.
