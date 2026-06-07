---
id: 019e8ca9-5093-76eb-9dbd-4b7a53ccf366
name: knowledge-hub
aliases:
  - kh
  - knowledge hub
  - knowledge-hub
updated_at: '2026-06-07T07:59:07.481Z'
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
knowledge-hub is a Model Context Protocol (MCP) server that captures Claude Code conversations into a personal knowledge vault. It runs a two-stage pipeline: Stage 1 archives the raw conversation (markdown + database row), and Stage 2 enqueues an extract job that mines reusable knowledge entities from the transcript.

> [!note] SQLite is the source of truth; the markdown vault is a generated view rendered by `kh sync`.

## Architecture
- **Storage:** SQLite (`.kh.db`) as the canonical store and Stage 2 job queue — deliberately avoids [[Redis]] since this is a local single-user tool. Redis would only make sense for shared, networked queues across multiple processes/machines.
- **Vault:** markdown files generated from the DB, carrying a "Generated from .kh.db" banner.

## Notes
- Exposed to Claude via the `archive_conversation` MCP tool.
- Stage 2 extract jobs are drained in-process by the server, or by a standalone `kh worker`.
- Reconnection can be triggered with `/mcp` in Claude Code; a successful reconnect reports `Reconnected to knowledge-hub.`
- End-to-end health check: invoke `archive_conversation` and verify both the archive write and the extract job enqueue succeed (e.g. conversation `019e8ca9-132d-721a-873d-aed9489849ec` as a recent smoke test).

## `kh sync` (DB → markdown renderer)
Materializes knowledge entities from `.kh.db` into vault markdown files and git-commits them.

Pipeline:
1. Finds dirty entities: `WHERE updated_at > synced_at AND deleted_at IS NULL` (plus deletions to remove files for).
2. Drift check first: compares file hash against `rendered_files.last_rendered_hash`; hand-edits are staged under `_proposals/manual_edit/` instead of being clobbered (review with `kh apply-proposal`).
3. Renders each entity to markdown with the "Generated from .kh.db" banner.
4. Bumps `synced_at` per file immediately after each write (not per batch) — a crash mid-render is resumable.
5. Git commits the result.

Flags (`src/bin/cli.ts:393`):
- `kh sync` — render all dirty entities → md + git commit
- `kh sync --entity NAME` — force one entity dirty and render
- `kh sync --since YYYY-MM-DD` — everything updated since date
- `kh sync --full` — clear `synced_at` on ALL alive entities → full rebuild
- `kh sync --dry-run` — show what would change

> [!tip] All flag modes work by clearing `synced_at` so the normal dirty query picks them up — one code path. Fails fast on errors; idempotency comes from the per-file `synced_at` bump, so re-running is always safe.

---

## 한국어

### 개요
knowledge-hub는 Claude Code 대화를 개인 지식 볼트로 캡처하는 Model Context Protocol (MCP) 서버다. 2단계 파이프라인으로 동작한다: Stage 1은 원본 대화를 아카이브하고(마크다운 + DB 행), Stage 2는 트랜스크립트에서 재사용 가능한 지식 엔티티를 추출하는 잡을 큐에 넣는다.

> [!note] SQLite가 진실의 원천(source of truth)이고, 마크다운 볼트는 `kh sync`로 렌더링되는 생성된 뷰다.

### 아키텍처
- **저장소:** SQLite (`.kh.db`)가 정식 저장소이자 Stage 2 잡 큐 — 로컬 단일 사용자 도구이므로 [[Redis]]를 의도적으로 쓰지 않는다. Redis는 여러 프로세스/머신에 걸친 공유 네트워크 큐가 필요할 때 의미가 있다.
- **볼트:** DB에서 생성된 마크다운 파일들로, "Generated from .kh.db" 배너를 달고 있다.

### 노트
- `archive_conversation` MCP 도구를 통해 Claude에 노출된다.
- Stage 2 추출 잡은 서버 인프로세스 또는 독립 `kh worker`로 처리된다.
- Claude Code에서 `/mcp`로 재연결을 트리거할 수 있다; 성공 시 `Reconnected to knowledge-hub.`가 보고된다.
- E2E 헬스 체크: `archive_conversation`을 호출해 아카이브 쓰기와 추출 잡 큐잉이 모두 성공하는지 확인한다 (최근 스모크 테스트 예: 대화 `019e8ca9-132d-721a-873d-aed9489849ec`).

### `kh sync` (DB → 마크다운 렌더러)
`.kh.db`의 지식 엔티티를 볼트 마크다운 파일로 머터리얼라이즈하고 git 커밋한다.

파이프라인:
1. 더티 엔티티를 찾는다: `WHERE updated_at > synced_at AND deleted_at IS NULL` (삭제할 파일 처리도 포함).
2. 드리프트 체크 먼저: 파일 해시를 `rendered_files.last_rendered_hash`와 비교; 수동 편집은 덮어쓰지 않고 `_proposals/manual_edit/` 아래에 스테이지된다 (`kh apply-proposal`로 검토).
3. 각 엔티티를 "Generated from .kh.db" 배너와 함께 마크다운으로 렌더링.
4. 배치 단위가 아닌 **파일 단위**로 매 쓰기 직후 `synced_at`을 갱신 — 렌더링 중 크래시가 나도 재개 가능.
5. 결과를 git 커밋.

플래그 (`src/bin/cli.ts:393`):
- `kh sync` — 모든 더티 엔티티 렌더링 → md + git 커밋
- `kh sync --entity NAME` — 특정 엔티티를 더티로 강제하고 렌더링
- `kh sync --since YYYY-MM-DD` — 해당 날짜 이후 갱신된 모두
- `kh sync --full` — 살아있는 모든 엔티티의 `synced_at`을 비워 전체 재빌드
- `kh sync --dry-run` — 변경될 내용 미리보기

> [!tip] 모든 플래그 모드는 `synced_at`을 비워 일반 더티 쿼리가 잡아내도록 하는 단일 코드 경로다. 에러 시 빠르게 실패하지만, 파일 단위 `synced_at` 갱신 덕분에 재실행이 항상 안전하다.

## Sources

- [[raw/conversations/019e8ca9-132d-721a-873d-aed9489849ec|019e8ca9-132d-721a-873d-aed9489849ec]]
- [[raw/conversations/019e8cae-3d03-76e8-8213-83715aec185d|019e8cae-3d03-76e8-8213-83715aec185d]]
