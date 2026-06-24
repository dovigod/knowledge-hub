---
id: 019e9be3-cf8d-725d-b0bc-81c36ef50c8c
name: tmux
aliases:
  - Terminal Multiplexer
  - multiplexer
  - terminal multiplexer
  - tmux
  - 터미널 멀티플렉서
  - 텀ux
updated_at: '2026-06-06T07:44:46.127Z'
summary: >-
  Terminal multiplexer that manages multiple terminal sessions, windows, and
  panes within a single SSH connection or terminal.
sources:
  - 019e9be2-e8f3-7388-975a-2600c07b1b31
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# tmux

## Overview

`tmux` is a terminal multiplexer that lets you manage multiple terminals inside a single SSH session or terminal window. It runs as a background server and organizes work into a clean hierarchy: **Server → Session → Window → Pane**. Sessions survive disconnects, so you can detach and reattach without losing running processes.

> [!note] Core hierarchy
> ```text
> tmux Server
> │
> ├── Session (e.g. a project)
> │   ├── Window (e.g. a role)
> │   │   ├── Pane
> │   │   └── Pane
> │   └── Window
> │       └── Pane
> └── Session
>     └── Window
>         └── Pane
> ```

## Notes

### Server

When you first run `tmux`, a background server process is created. All sessions live under this server.

```bash
tmux
ps aux | grep tmux
```

### Session

The top-level work unit — typically one session per project.

```bash
tmux new -s backend
tmux new -s frontend
tmux new -s monitoring
tmux ls
```

```text
tmux server
├── backend
├── frontend
└── monitoring
```

Sessions are independent workspaces and survive SSH disconnection. Common split by purpose:

- `backend` → API server
- `frontend` → React development
- `monitoring` → log monitoring

### Window

Like a browser tab inside a session. Often used per role (editor, logs, shell).

```text
Session: backend

Window 0: editor
Window 1: logs
Window 2: shell
```

> [!tip] Window shortcuts
> - Create: `Ctrl+b c`
> - Next: `Ctrl+b n`
> - Previous: `Ctrl+b p`

### Pane

A pane is a split inside a window — an actual running terminal.

```text
+------------------+
| editor           |
+---------+--------+
| build   | logs   |
+---------+--------+
```

> [!tip] Pane shortcuts
> - Horizontal split: `Ctrl+b "`
> - Vertical split: `Ctrl+b %`
> - Move between panes: `Ctrl+b <arrow>`

### Attach / Detach

> [!example] Detach and reattach
> ```bash
> # Detach (session keeps running)
> Ctrl+b d
>
> # Reattach later
> tmux attach -t backend
> ```

The session stays alive on the server; only the terminal connection is detached.

### Common convention

- **Project** → Session
- **Role** (editor / logs / build) → Window
- **Individual running terminal** → Pane

Related: [[SSH]], [[Terminal]], [[screen]], [[shell]]

---

## 한국어

### 개요

`tmux`는 터미널 멀티플렉서(Terminal Multiplexer)로, 하나의 SSH 세션이나 터미널 안에서 여러 터미널을 관리할 수 있게 해준다. 백그라운드 서버 프로세스로 실행되며 **Server → Session → Window → Pane**의 깔끔한 계층 구조로 작업을 조직한다. 세션은 연결이 끊겨도 살아있어서 detach/attach로 작업을 이어갈 수 있다.

> [!note] 핵심 계층
> ```text
> tmux Server
> │
> ├── Session (예: 프로젝트)
> │   ├── Window (예: 역할)
> │   │   ├── Pane
> │   │   └── Pane
> │   └── Window
> │       └── Pane
> └── Session
>     └── Window
>         └── Pane
> ```

### 노트

#### Server

`tmux`를 처음 실행하면 백그라운드에 tmux 서버 프로세스가 생성된다. 모든 세션은 이 서버 아래에 존재한다.

```bash
tmux
ps aux | grep tmux
```

#### Session

가장 상위 작업 단위 — 보통 프로젝트 하나당 세션 하나로 구성한다.

```bash
tmux new -s backend
tmux new -s frontend
tmux new -s monitoring
tmux ls
```

```text
tmux server
├── backend
├── frontend
└── monitoring
```

세션은 독립적인 작업 공간이며 SSH가 끊겨도 살아있다. 용도별 구성 예시:

- `backend` → API 서버
- `frontend` → React 개발
- `monitoring` → 로그 모니터링

#### Window

세션 안의 탭(Tab) 개념. 역할별(editor, logs, shell)로 나누어 쓰는 경우가 많다.

```text
Session: backend

Window 0: editor
Window 1: logs
Window 2: shell
```

> [!tip] Window 단축키
> - 생성: `Ctrl+b c`
> - 다음: `Ctrl+b n`
> - 이전: `Ctrl+b p`

#### Pane

윈도우를 분할한 실제 터미널.

```text
+------------------+
| editor           |
+---------+--------+
| build   | logs   |
+---------+--------+
```

> [!tip] Pane 단축키
> - 수평 분할: `Ctrl+b "`
> - 수직 분할: `Ctrl+b %`
> - 패널 이동: `Ctrl+b <방향키>`

#### Attach / Detach

> [!example] Detach와 재접속
> ```bash
> # Detach (세션은 계속 실행됨)
> Ctrl+b d
>
> # 나중에 다시 접속
> tmux attach -t backend
> ```

세션은 서버에 그대로 유지되고 터미널 연결만 분리된다.

#### 실무 컨벤션

- **프로젝트** → Session
- **역할**(editor / logs / build) → Window
- **실행 중인 개별 터미널** → Pane

관련: [[SSH]], [[Terminal]], [[screen]], [[shell]]

## Sources

- [[raw/conversations/019e9be2-e8f3-7388-975a-2600c07b1b31|019e9be2-e8f3-7388-975a-2600c07b1b31]]
