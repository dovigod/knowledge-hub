# Smart Connections setup

This vault is paired with the [Smart Connections](https://github.com/brianpetro/obsidian-smart-connections)
Obsidian plugin. Smart Connections owns embeddings and the "related notes"
sidebar — the knowledge-hub itself does **not** read or write `.smart-env/`.

## Install

1. Open Obsidian with this vault.
2. Settings → Community plugins → Browse.
3. Search for **Smart Connections** (by Brian Petro).
4. Install → Enable.
5. Restart Obsidian if prompted.

## Recommended settings

| Setting | Value |
|---|---|
| Folders to include | `knowledge/` |
| Folders to exclude | `raw/`, `_proposals/`, `_backups/` |
| Index frontmatter | on (entity name, aliases, summary live there) |
| Smart View / Smart Chat | optional — user preference |

The canonical entity pages live under `knowledge/{Entity}.md` and are
re-rendered from `.kh.db` by `kh sync`. Pointing Smart Connections at
that folder gives the best signal: short, single-concept pages with tags
and aliases in frontmatter.

The raw conversation log under `raw/conversations/` is high-volume and
low-signal for semantic discovery; exclude it.

## Where state lives

- `.smart-env/` — Smart Connections plugin state (vectors, settings). Already
  ignored by `.gitignore`. The hub never reads this.
- `knowledge/{Entity}.md` — canonical entity pages. Smart Connections indexes
  these.

## Verifying it works

1. Run a few `archive_conversation` calls and `kh sync`.
2. Open any `knowledge/{Entity}.md` in Obsidian.
3. Open the Smart Connections sidebar (ribbon icon or command palette
   "Smart Connections: Open Smart View").
4. Related entities should surface automatically. If not, give Smart
   Connections time to finish its initial index pass (the status bar
   shows progress).

## Contract

The hub does not depend on Smart Connections for correctness. If you
uninstall the plugin, the vault still renders; you only lose the
related-notes UX and fall back to Obsidian's native backlinks + tag
search.
