---
id: 019e8ca9-5096-7260-96f5-4a81dbbf977b
name: Model Context Protocol
aliases:
  - MCP
  - mcp
  - mcp protocol
updated_at: '2026-06-03T08:46:06.998Z'
summary: >-
  An open protocol that lets LLM clients like Claude Code connect to external
  tool/data servers over a standardized interface.
sources:
  - 019e8ca9-132d-721a-873d-aed9489849ec
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Model Context Protocol

## Overview
The Model Context Protocol (MCP) is the standard interface Claude Code uses to talk to external servers that expose tools, resources, and prompts. MCP servers can be local processes or remote HTTP/SSE endpoints, and they appear to the model as callable tools namespaced by server name.

## Notes
- Claude Code manages MCP servers via the `/mcp` command (list, reconnect, authenticate).
- Reconnecting an MCP server re-establishes the tool surface without restarting the Claude session.
- Each MCP server's tools are prefixed (e.g. `mcp__knowledge-hub__archive_conversation`).

## Sources

- [[raw/conversations/019e8ca9-132d-721a-873d-aed9489849ec|019e8ca9-132d-721a-873d-aed9489849ec]]
