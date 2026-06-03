---
id: 019e8cae-a6c9-7548-a060-78d1c6be50e9
name: kh sync
aliases:
  - kh-sync
updated_at: '2026-06-03T08:51:56.746Z'
summary: >-
  Knowledge-hub CLI command that renders dirty entities from SQLite (.kh.db)
  into the markdown vault and git-commits the result.
sources:
  - 019e8cae-3d03-76e8-8213-83715aec185d
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# kh sync

## Overview

`kh sync` is the [[SQLite]] → markdown renderer for [[knowledge-hub]]. It materializes knowledge entities from `.kh.db` into the markdown vault files and git-commits them. SQLite is the source of truth; the `.md` files are a generated view.

## Notes

### Pipeline
1. **Find dirty entities**: `WHERE updated_at > synced_at AND deleted_at IS NULL` (plus deleted entities to remove their files).
2. **Drift check**: compares the file hash against `rendered_files.last_rendered_hash`; hand-edits are staged under `_proposals/manual_edit/` instead of being clobbered (review with `kh apply-proposal`).
3. **Render** each entity to markdown with the "Generated from .kh.db" banner.
4. **Bump `synced_at` per file** immediately after each write (not per batch), so a crash mid-render is resumable.
5. **Git commit** the result.

### Flags (src/bin/cli.ts:393)
- `kh sync` — render all dirty entities → md + git commit
- `kh sync --entity NAME` — force one entity dirty and render
- `kh sync --since YYYY-MM-DD` — everything updated since date
- `kh sync --full` — clear `synced_at` on ALL alive entities → full rebuild
- `kh sync --dry-run` — show what would change

### Design notes
- All flag modes work by clearing `synced_at` so the normal dirty query picks them up — one code path.
- **Failure mode**: fails fast with the underlying error; idempotency comes from the per-file `synced_at` bump, so re-running is always safe.

## Sources

- [[raw/conversations/019e8cae-3d03-76e8-8213-83715aec185d|019e8cae-3d03-76e8-8213-83715aec185d]]
