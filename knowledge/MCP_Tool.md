---
id: 019e8cb6-0a19-741e-a402-d579d44b9d9a
name: MCP Tool
aliases:
  - mcp tool
  - tool (mcp)
  - tools/call
  - tools/list
updated_at: '2026-06-03T09:00:00.921Z'
summary: >-
  An executable function exposed by an MCP server to LLMs, defined by a name,
  description, and JSON Schema input contract.
sources:
  - 019e8cb5-80c2-71ae-b27a-8f34e3216d7d
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# MCP Tool

## Overview

In [[Model Context Protocol]], a **Tool** is an executable function that an MCP server exposes to an LLM. When the LLM decides to invoke it, the server runs real code and returns the result. Tools are *model-controlled* — the LLM reads each tool's description and chooses when and how to call it.

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

### Example in knowledge-hub

`src/bin/mcp-server.ts` exposes `archive_conversation` as a tool:
- **Schema**: `source`, `messages` (required), plus optional `topics`, `tags`, `git`, etc.
- **Behavior**: writes raw markdown to the vault, inserts a `conversations` row in SQLite, enqueues a Stage 2 extract job, then returns JSON containing `conversation_id`, `path`, and `extract_job_id`.

## Sources

- [[raw/conversations/019e8cb5-80c2-71ae-b27a-8f34e3216d7d|019e8cb5-80c2-71ae-b27a-8f34e3216d7d]]
