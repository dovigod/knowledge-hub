---
id: 019e8ca9-5093-76eb-9dbd-4b7a53ccf366
name: knowledge-hub
aliases:
  - kh
  - knowledge hub
updated_at: '2026-06-03T08:46:06.995Z'
summary: >-
  A local MCP server that archives Claude Code conversations and extracts
  reusable knowledge entities from them.
sources:
  - 019e8ca9-132d-721a-873d-aed9489849ec
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# knowledge-hub

## Overview
knowledge-hub is a Model Context Protocol (MCP) server that captures Claude Code conversations into a personal knowledge vault. It runs a two-stage pipeline: Stage 1 archives the raw conversation (markdown + database row), and Stage 2 enqueues an extract job that mines reusable knowledge entities from the transcript.

## Notes
- Exposed to Claude via the `archive_conversation` MCP tool.
- Stage 2 extract jobs are drained in-process by the server, or by a standalone `kh worker`.
- Reconnection can be triggered with `/mcp` in Claude Code; a successful reconnect reports `Reconnected to knowledge-hub.`
- End-to-end health check: invoke `archive_conversation` and verify both the archive write and the extract job enqueue succeed.

## Sources

- [[raw/conversations/019e8ca9-132d-721a-873d-aed9489849ec|019e8ca9-132d-721a-873d-aed9489849ec]]
