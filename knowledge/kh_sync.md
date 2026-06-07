---
id: 019e8cae-a6c9-7548-a060-78d1c6be50e9
name: kh sync
aliases:
  - kh sync
  - kh-sync
  - sync command
updated_at: '2026-06-07T07:58:31.185Z'
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

> [!note] Source of truth
> Edit the DB, not the markdown. Hand-edits to generated files are caught by the drift check and staged for review instead of being silently overwritten.

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

> [!tip] Resumability
> The per-file `synced_at` bump (not per-batch) means a crash mid-render leaves already-rendered files marked clean — just re-run `kh sync`.

---

## 한국어

### 개요

`kh sync`는 [[knowledge-hub]]의 [[SQLite]] → 마크다운 렌더러입니다. `.kh.db`에 저장된 knowledge entity들을 마크다운 vault 파일로 만들어내고 git 커밋까지 수행합니다. SQLite가 source of truth이고, `.md` 파일은 생성된 view입니다.

> [!note] Source of truth
> 마크다운이 아니라 DB를 수정하세요. 생성된 파일을 직접 손대면 drift check가 잡아내어 조용히 덮어쓰지 않고 리뷰 대기열에 올려둡니다.

### 노트

#### 파이프라인
1. **Dirty entity 탐색**: `WHERE updated_at > synced_at AND deleted_at IS NULL` (삭제된 entity의 파일 제거 포함).
2. **Drift check**: 파일 해시와 `rendered_files.last_rendered_hash`를 비교. 손으로 수정된 부분은 덮어쓰지 않고 `_proposals/manual_edit/` 아래에 staging (검토는 `kh apply-proposal`).
3. **렌더링**: 각 entity를 "Generated from .kh.db" 배너와 함께 마크다운으로 출력.
4. **파일별 `synced_at` 갱신**: 배치 단위가 아니라 파일 쓰기 직후마다 갱신 → 렌더링 도중 크래시가 나도 resumable.
5. **Git 커밋**: 결과물을 커밋.

#### 플래그 (src/bin/cli.ts:393)
- `kh sync` — 모든 dirty entity 렌더링 → md + git 커밋
- `kh sync --entity NAME` — 특정 entity 하나를 강제로 dirty 처리 후 렌더링
- `kh sync --since YYYY-MM-DD` — 해당 날짜 이후 업데이트된 모든 entity
- `kh sync --full` — 살아있는 모든 entity의 `synced_at`을 clear → 전체 재빌드
- `kh sync --dry-run` — 무엇이 바뀔지 미리보기

#### 설계 노트
- 모든 플래그 모드는 `synced_at`을 비워서 평소의 dirty 쿼리가 잡도록 하는 방식 — 코드 경로는 하나.
- **실패 동작**: 에러가 나면 그 즉시 fail fast. 멱등성은 파일별 `synced_at` 갱신으로 확보되므로 재실행은 언제나 안전.

> [!tip] 재실행 안전성
> `synced_at` 갱신이 배치가 아닌 파일 단위라, 렌더링 도중 크래시가 나도 이미 처리된 파일은 clean으로 표시되어 있음 — 그냥 `kh sync`를 다시 실행하면 됩니다.

## Sources

- [[raw/conversations/019e8cae-3d03-76e8-8213-83715aec185d|019e8cae-3d03-76e8-8213-83715aec185d]]
