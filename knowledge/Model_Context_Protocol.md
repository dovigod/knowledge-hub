---
id: 019e8ca9-5096-7260-96f5-4a81dbbf977b
name: Model Context Protocol
aliases:
  - MCP
  - Model Context Protocol
  - mcp
  - mcp protocol
  - model-context-protocol
updated_at: '2026-06-07T07:58:14.965Z'
summary: >-
  An open protocol that lets LLM clients like Claude Code connect to external
  tool/data servers over a standardized interface.
sources:
  - 019e8ca9-132d-721a-873d-aed9489849ec
  - 019e8cb5-80c2-71ae-b27a-8f34e3216d7d
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Model Context Protocol

## Overview
The Model Context Protocol (MCP) is the standard interface Claude Code uses to talk to external servers that expose tools, resources, and prompts. MCP servers can be local processes or remote HTTP/SSE endpoints, and they appear to the model as callable tools namespaced by server name.

> [!note] Tools are the model-facing primitive
> When a user asks "what is a tool in MCP?", the short answer is: a named function with a JSON Schema that the LLM can call autonomously. The server executes it; the result flows back to the model.

## Primitives
MCP servers expose three primitives, distinguished by who decides when they run:
- **Tools** — model-controlled. The LLM reads each tool's name, description, and JSON Schema, then decides when to call it. Calls can have side effects.
- **Resources** — application-controlled. Read-only data the host surfaces as context.
- **Prompts** — user-controlled. Templates the user picks explicitly (slash-command style).

## Tool call flow
1. Client sends `tools/list`; the server returns each tool's name, description, and input schema.
2. The LLM decides to invoke one and sends `tools/call` with the tool name and arguments.
3. The server executes the underlying function and returns the result to the LLM.

The LLM picks the tool from its description alone, so descriptions and schemas are the contract.

## Notes
- Claude Code manages MCP servers via the `/mcp` command (list, reconnect, authenticate).
- Reconnecting an MCP server re-establishes the tool surface without restarting the Claude session.
- Each MCP server's tools are prefixed (e.g. `mcp__knowledge-hub__archive_conversation`).

> [!tip] Description quality is the contract
> Because the LLM only sees the tool's description and schema, vague descriptions cause the wrong tool to be picked — or none at all.

## Examples
`src/bin/mcp-server.ts` in [[knowledge-hub]] exposes `archive_conversation` as a tool:
- **Schema:** `source` and `messages` required; optional `topics`, `tags`, `git`, etc.
- **Execution:** writes the raw markdown to the vault, inserts a `conversations` row in SQLite, enqueues a Stage 2 extract job, and returns `{ conversation_id, path, extract_job_id }`.

> [!example] Self-referential archive
> Archiving the very conversation that asks "what is a tool in MCP?" is itself an instance of the tool-call flow above — the model reads `archive_conversation`'s schema, calls it, and the server writes the markdown plus enqueues the extract job.

---

## 한국어

### 개요
Model Context Protocol(MCP)은 Claude Code가 도구, 리소스, 프롬프트를 노출하는 외부 서버와 통신하는 표준 인터페이스다. MCP 서버는 로컬 프로세스이거나 원격 HTTP/SSE 엔드포인트일 수 있고, 모델 입장에서는 서버 이름으로 네임스페이스된 호출 가능한 도구로 보인다.

> [!note] Tool은 모델이 직접 다루는 primitive
> "MCP의 tool이 뭐야?"라는 질문의 짧은 답은: JSON Schema가 붙은 이름 있는 함수이며, LLM이 스스로 판단해 호출할 수 있는 것. 서버가 실행하고, 결과가 모델로 돌아온다.

### Primitives
MCP 서버가 노출하는 세 가지 primitive — 누가 실행 시점을 결정하는지로 구분된다:
- **Tools** — 모델이 제어. LLM이 각 tool의 이름, 설명, JSON Schema를 읽고 언제 호출할지 스스로 결정. 부작용을 가질 수 있다.
- **Resources** — 애플리케이션이 제어. 호스트가 컨텍스트로 노출하는 읽기 전용 데이터.
- **Prompts** — 사용자가 제어. 슬래시 커맨드처럼 사용자가 명시적으로 고르는 템플릿.

### Tool 호출 흐름
1. 클라이언트가 `tools/list`를 보내면, 서버가 각 tool의 이름, 설명, 입력 스키마를 반환.
2. LLM이 호출을 결정하고 `tools/call`을 tool 이름과 인자와 함께 전송.
3. 서버가 실제 함수를 실행하고 결과를 LLM에 반환.

LLM은 오직 설명만 보고 tool을 고르므로, 설명과 스키마가 곧 계약이다.

### 노트
- Claude Code는 `/mcp` 커맨드로 MCP 서버를 관리한다 (목록, 재연결, 인증).
- MCP 서버를 재연결하면 Claude 세션을 재시작하지 않고도 tool surface가 다시 살아난다.
- 각 MCP 서버의 tool은 prefix가 붙는다 (예: `mcp__knowledge-hub__archive_conversation`).

> [!tip] 설명의 품질이 곧 계약
> LLM은 tool의 설명과 스키마만 보기 때문에, 설명이 모호하면 엉뚱한 tool이 선택되거나 아예 선택되지 않는다.

### 예시
[[knowledge-hub]]의 `src/bin/mcp-server.ts`는 `archive_conversation`을 tool로 노출한다:
- **스키마:** `source`와 `messages`는 필수, `topics`, `tags`, `git` 등은 선택.
- **실행:** raw 마크다운을 vault에 쓰고, SQLite의 `conversations` 테이블에 행을 삽입하고, Stage 2 extract 잡을 enqueue한 뒤 `{ conversation_id, path, extract_job_id }`를 반환한다.

> [!example] 자기 참조적인 아카이브
> "MCP의 tool이 뭐야?"라고 물은 그 대화 자체를 아카이브하는 행위가 위의 tool 호출 흐름의 한 사례 — 모델이 `archive_conversation`의 스키마를 읽고 호출하면, 서버가 마크다운을 쓰고 extract 잡을 enqueue한다.

## Sources

- [[raw/conversations/019e8ca9-132d-721a-873d-aed9489849ec|019e8ca9-132d-721a-873d-aed9489849ec]]
- [[raw/conversations/019e8cb5-80c2-71ae-b27a-8f34e3216d7d|019e8cb5-80c2-71ae-b27a-8f34e3216d7d]]
