---
id: 019e8cb6-0a19-741e-a402-d579d44b9d9a
name: MCP Tool
aliases:
  - MCP Tool
  - mcp tool
  - mcp-tool
  - model context protocol tool
  - tool (mcp)
  - tools/call
  - tools/list
updated_at: '2026-06-07T07:57:42.369Z'
summary: >-
  An executable function exposed by an MCP server to LLMs, defined by a name,
  description, and JSON Schema input contract.
sources:
  - 019e8cb5-80c2-71ae-b27a-8f34e3216d7d
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# MCP Tool

## Overview

In [[Model Context Protocol]], a **Tool** is an executable function that an MCP server exposes to an LLM. When the LLM decides to invoke it, the server runs real code and returns the result.

> [!note] Tools are *model-controlled*
> The LLM reads each tool's description and chooses when and how to call it — unlike [[MCP Resource|Resources]] (app-controlled) or [[MCP Prompt|Prompts]] (user-controlled).

## Notes

### Anatomy

A tool is defined by:
- **name** — stable identifier used in `tools/call`
- **description** — natural-language hint the LLM uses to decide when to invoke it
- **input schema** — JSON Schema describing accepted arguments

### Call Flow

1. Client sends `tools/list` → server returns the catalog of tools with their schemas.
2. LLM decides to invoke a tool → client sends `tools/call` with the tool name and arguments.
3. Server executes the underlying function → returns the result to the LLM.

### Tools vs Resources vs Prompts

| Primitive | Controlled by | Purpose |
|-----------|---------------|---------|
| Tools     | LLM (model)   | Actions with possible side effects |
| Resources | Application   | Read-only context data |
| Prompts   | User          | Reusable templates / slash commands |

> [!tip] Side effects are expected
> Because tools can mutate state (write files, call APIs, run queries), descriptions and schemas matter — they're the LLM's only signal for *when* it's safe to invoke.

## Examples

### `archive_conversation` in knowledge-hub

`src/bin/mcp-server.ts` exposes `archive_conversation` as a tool:
- **Schema**: `source`, `messages` (required), plus optional `topics`, `tags`, `git`, etc.
- **Behavior**: writes raw markdown to the vault, inserts a `conversations` row in SQLite, enqueues a Stage 2 extract job, then returns JSON containing `conversation_id`, `path`, and `extract_job_id`.

> [!example] End-to-end
> The LLM sees `archive_conversation`'s description, decides this chat is worth saving, calls it with the message list, and the server handles persistence + indexing autonomously.

---

## 한국어

### 개요

[[Model Context Protocol]]에서 **Tool**은 MCP 서버가 LLM에게 노출하는 실행 가능한 함수다. LLM이 호출하기로 결정하면 서버가 실제 코드를 실행하고 결과를 돌려준다.

> [!note] Tool은 *model-controlled*
> LLM이 각 tool의 description을 읽고 언제, 어떻게 호출할지 스스로 결정한다 — [[MCP Resource|Resource]] (앱이 제어) 또는 [[MCP Prompt|Prompt]] (사용자가 제어)와 다른 점.

### 노트

#### 구성 요소

Tool은 다음으로 정의된다:
- **name** — `tools/call`에서 쓰이는 안정적인 식별자
- **description** — LLM이 호출 시점을 판단하는 자연어 힌트
- **input schema** — 인자를 기술하는 JSON Schema

#### 호출 흐름

1. 클라이언트가 `tools/list` 전송 → 서버가 스키마와 함께 tool 카탈로그 반환.
2. LLM이 tool 호출 결정 → 클라이언트가 tool 이름과 인자로 `tools/call` 전송.
3. 서버가 실제 함수 실행 → 결과를 LLM에 반환.

#### Tool vs Resource vs Prompt

| Primitive | 제어 주체     | 용도 |
|-----------|---------------|------|
| Tools     | LLM (모델)    | 부수효과가 있을 수 있는 액션 |
| Resources | 애플리케이션  | 읽기 전용 컨텍스트 데이터 |
| Prompts   | 사용자        | 재사용 가능한 템플릿 / 슬래시 커맨드 |

> [!tip] 부수효과가 전제됨
> Tool은 파일 쓰기, API 호출, 쿼리 실행 등 상태를 변경할 수 있기 때문에 description과 schema가 중요하다 — LLM이 *언제* 안전하게 호출할지 판단하는 유일한 신호다.

### 예시

#### knowledge-hub의 `archive_conversation`

`src/bin/mcp-server.ts`는 `archive_conversation`을 tool로 노출한다:
- **스키마**: `source`, `messages` (필수), 그리고 선택적 `topics`, `tags`, `git` 등.
- **동작**: vault에 raw markdown을 쓰고, SQLite의 `conversations` 행을 삽입하고, Stage 2 extract job을 enqueue한 뒤, `conversation_id`, `path`, `extract_job_id`를 담은 JSON을 반환한다.

> [!example] 엔드 투 엔드
> LLM이 `archive_conversation`의 description을 보고 이 대화를 저장할 가치가 있다고 판단하면, 메시지 목록과 함께 호출하고 서버가 알아서 영속화와 인덱싱을 처리한다.

## Sources

- [[raw/conversations/019e8cb5-80c2-71ae-b27a-8f34e3216d7d|019e8cb5-80c2-71ae-b27a-8f34e3216d7d]]
