---
id: 019e8ca9-5093-76eb-9dbd-4b7a53ccf366
name: knowledge-hub
aliases:
  - kh
  - knowledge hub
  - knowledge-hub
updated_at: '2026-06-03T08:52:14.022Z'
summary: >-
  A local MCP server that archives Claude Code conversations and extracts
  reusable knowledge entities from them.
sources:
  - 019e8ca9-132d-721a-873d-aed9489849ec
  - 019e8cae-3d03-76e8-8213-83715aec185d
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# knowledge-hub

## Overview
knowledge-hub is a Model Context Protocol (MCP) server that captures Claude Code conversations into a personal knowledge vault. It runs a two-stage pipeline: Stage 1 archives the raw conversation (markdown + database row), and Stage 2 enqueues an extract job that mines reusable knowledge entities from the transcript. SQLite is the source of truth; the markdown vault is a generated view rendered by `kh sync`.

## Architecture
- **Storage:** SQLite (`.kh.db`) as the canonical store and Stage 2 job queue — deliberately avoids Redis since this is a local single-user tool. Redis would only make sense for shared, networked queues across multiple processes/machines.
- **Vault:** markdown files generated from the DB, carrying a "Generated from .kh.db" banner.

## Notes
- Exposed to Claude via the `archive_conversation` MCP tool.
- Stage 2 extract jobs are drained in-process by the server, or by a standalone `kh worker`.
- Reconnection can be triggered with `/mcp` in Claude Code; a successful reconnect reports `Reconnected to knowledge-hub.`
- End-to-end health check: invoke `archive_conversation` and verify both the archive write and the extract job enqueue succeed.

## `kh sync` (DB → markdown renderer)
Materializes knowledge entities from `.kh.db` into vault markdown files and git-commits them.

Pipeline:
1. Finds dirty entities: `WHERE updated_at > synced_at AND deleted_at IS NULL` (plus deletions to remove files for).
2. Drift check first: compares file hash against `rendered_files.last_rendered_hash`; hand-edits are staged under `_proposals/manual_edit/` instead of being clobbered (review with `kh apply-proposal`).
3. Renders each entity to markdown.
4. Bumps `synced_at` per file immediately after each write (not per batch) — a crash mid-render is resumable.
5. Git commits the result.

Flags (`src/bin/cli.ts:393`):
- `kh sync` — render all dirty entities → md + git commit
- `kh sync --entity NAME` — force one entity dirty and render
- `kh sync --since YYYY-MM-DD` — everything updated since date
- `kh sync --full` — clear `synced_at` on ALL alive entities → full rebuild
- `kh sync --dry-run` — show what would change

All flag modes work by clearing `synced_at` so the normal dirty query picks them up — one code path. Fails fast on errors; idempotency comes from the per-file `synced_at` bump, so re-running is always safe.

## Sources

- [[raw/conversations/019e8ca9-132d-721a-873d-aed9489849ec|019e8ca9-132d-721a-873d-aed9489849ec]]
- [[raw/conversations/019e8cae-3d03-76e8-8213-83715aec185d|019e8cae-3d03-76e8-8213-83715aec185d]]
