---
id: 019ea116-d359-7582-8187-eb1d26efefa5
title: MCP Tool 개념과 호출 흐름
topics:
  - mcp
  - tool
  - model-context-protocol
  - llm
sources:
  - 019e8cb5-80c2-71ae-b27a-8f34e3216d7d
created_at: '2026-06-07T07:58:08.217Z'
updated_at: '2026-06-07T07:58:08.217Z'
---
## What is a Tool in MCP?

A [[Tool]] is an executable function that a server exposes to an [[LLM]] in [[MCP]] (Model Context Protocol). When the LLM invokes it, the server runs the actual code and returns the result.

### Core Concepts
- Definition: name + description + input schema ([[JSON Schema]]). The LLM reads the description and decides on its own when to use which tool.
- Call flow: (1) the client sends a `tools/list` request → the server returns the tool list and schemas, (2) the LLM decides to call a tool → sends a `tools/call` request (name + arguments), (3) the server executes the function → returns the result to the LLM.
- How model-controlled differs from the other primitives:
  - Tools — invoked at the LLM's discretion (actions, side effects possible)
  - Resources — read-only data provided by the application as context
  - Prompts — templates the user selects (like slash commands)

### Example in knowledge-hub
`archive_conversation`, exposed by `src/bin/mcp-server.ts`, is exactly a tool:
- Name: `archive_conversation`
- Schema: `source`, `messages` (required), `topics`, `tags`, `git`, etc.
- Execution: when invoked, writes raw md into the vault, inserts a `conversations` row into [[SQLite]], enqueues a Stage 2 extract job, then returns a result JSON (`conversation_id`, `path`, `extract_job_id`).

The act of archiving this very conversation is itself precisely an example of this mechanism.

---

## 한국어

## MCP에서 Tool이란?

[[Tool]]은 [[MCP]](Model Context Protocol)에서 서버가 [[LLM]]에게 노출하는 실행 가능한 함수입니다. LLM이 호출하면 서버가 실제 코드를 실행하고 결과를 돌려줍니다.

### 핵심 개념
- 정의: 이름 + 설명 + 입력 스키마([[JSON Schema]]). LLM은 설명을 읽고 언제 어떤 tool을 쓸지 스스로 판단.
- 호출 흐름: (1) 클라이언트가 `tools/list` 요청 → 서버가 tool 목록과 스키마 반환, (2) LLM이 tool 호출 결정 → `tools/call` 요청(이름+인자), (3) 서버가 함수 실행 → 결과를 LLM에게 반환.
- 모델 주도(model-controlled)가 다른 프리미티브와의 차이:
  - Tools — LLM이 판단해서 호출 (행동, side effect 가능)
  - Resources — 애플리케이션이 컨텍스트로 제공하는 읽기 전용 데이터
  - Prompts — 사용자가 선택하는 템플릿 (슬래시 커맨드처럼)

### knowledge-hub에서의 예시
`src/bin/mcp-server.ts`가 노출하는 `archive_conversation`이 바로 tool입니다:
- 이름: `archive_conversation`
- 스키마: `source`, `messages` (필수), `topics`, `tags`, `git` 등
- 실행: 호출되면 raw md를 vault에 쓰고, [[SQLite]]에 `conversations` row를 넣고, Stage 2 extract job을 큐에 넣은 뒤 결과 JSON(`conversation_id`, `path`, `extract_job_id`)을 반환

이 대화를 아카이브하는 것 자체가 정확히 이 메커니즘의 예시입니다.
