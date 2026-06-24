---
id: 019ea116-13c6-7248-ad9b-bdbf4ea19788
title: kh sync 동작 원리
topics:
  - knowledge-hub
  - kh
  - sync
  - sqlite
  - markdown 렌더링
sources:
  - 019e8cae-3d03-76e8-8213-83715aec185d
created_at: '2026-06-07T07:57:19.174Z'
updated_at: '2026-06-07T07:57:19.174Z'
---
## What `kh sync` does

`kh sync` is the SQLite → markdown renderer — it materializes knowledge entities from [[.kh.db]] into the markdown vault files and git-commits them. [[SQLite]] is the source of truth; the .md files are a generated view.

What it does:
1. Finds dirty entities: entities WHERE `updated_at > synced_at AND deleted_at IS NULL` (plus deleted entities to remove files for)
2. Drift check first: compares file hash against `rendered_files.last_rendered_hash`; hand-edits are staged under `_proposals/manual_edit/` instead of being clobbered (review with `kh apply-proposal`)
3. Renders each entity to markdown with the "Generated from .kh.db" banner
4. Bumps `synced_at` per file immediately after each write (not per batch) so a crash mid-render is resumable
5. Git commits the result

## Flags

Flags (`src/bin/cli.ts:393`):
- `kh sync` — render all dirty entities → md + git commit
- `kh sync --entity NAME` — force one entity dirty and render
- `kh sync --since YYYY-MM-DD` — everything updated since date
- `kh sync --full` — clear `synced_at` on ALL alive entities → full rebuild
- `kh sync --dry-run` — show what would change

All flag modes work by clearing `synced_at` so the normal dirty query picks them up — one code path. Failure behavior: fails fast with the underlying error; idempotency comes from the per-file `synced_at` bump so re-running is always safe.

---

## 한국어

### `kh sync`가 하는 일

`kh sync`는 SQLite → markdown 렌더러로, [[.kh.db]]의 knowledge entity들을 마크다운 vault 파일로 구체화하고 git 커밋한다. [[SQLite]]가 source of truth이며, .md 파일은 생성된 view다.

하는 일:
1. dirty entity 찾기: `updated_at > synced_at AND deleted_at IS NULL`인 entity들 (그리고 파일을 제거할 deleted entity들)
2. drift check 먼저: 파일 해시를 `rendered_files.last_rendered_hash`와 비교; 수기 편집은 덮어쓰지 않고 `_proposals/manual_edit/` 아래에 staging 됨 (`kh apply-proposal`로 검토)
3. 각 entity를 "Generated from .kh.db" 배너와 함께 마크다운으로 렌더
4. 각 파일 쓰기 직후 (batch 단위가 아닌) `synced_at`을 즉시 갱신해서 렌더 중간에 crash가 나도 재개 가능
5. 결과를 git 커밋

### 플래그

플래그 (`src/bin/cli.ts:393`):
- `kh sync` — 모든 dirty entity를 렌더 → md + git 커밋
- `kh sync --entity NAME` — 한 entity를 강제로 dirty로 만들고 렌더
- `kh sync --since YYYY-MM-DD` — 지정 날짜 이후 업데이트된 모든 것
- `kh sync --full` — alive entity 전체의 `synced_at`을 비움 → 전체 재빌드
- `kh sync --dry-run` — 무엇이 바뀔지 보여줌

모든 플래그 모드는 `synced_at`을 비워서 일반 dirty query가 집어가게 하는 방식으로 동작 — 코드 경로 하나. 실패 동작: 내부 에러와 함께 fail-fast; idempotency는 파일 단위 `synced_at` 갱신에서 오므로 재실행은 항상 안전하다.
