---
id: 019e8ca9-5096-7260-96f5-4a81dbbf977b
name: Model Context Protocol
aliases:
  - MCP
  - Model Context Protocol
  - mcp
  - mcp protocol
updated_at: '2026-06-03T08:59:55.446Z'
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

## Examples
`src/bin/mcp-server.ts` in [[knowledge-hub]] exposes `archive_conversation` as a tool:
- **Schema:** `source` and `messages` required; optional `topics`, `tags`, `git`, etc.
- **Execution:** writes the raw markdown to the vault, inserts a `conversations` row in SQLite, enqueues a Stage 2 extract job, and returns `{ conversation_id, path, extract_job_id }`.

Archiving a conversation through this server is itself a live example of the tool-call flow above.

## Sources

- [[raw/conversations/019e8ca9-132d-721a-873d-aed9489849ec|019e8ca9-132d-721a-873d-aed9489849ec]]
- [[raw/conversations/019e8cb5-80c2-71ae-b27a-8f34e3216d7d|019e8cb5-80c2-71ae-b27a-8f34e3216d7d]]
