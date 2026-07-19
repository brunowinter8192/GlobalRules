# Tool-Use — Orchestrator-Only

## Bash

### Worker CLI

**Worker names are globally unique, registry-tracked.**
Project path is required only for `spawn`; other commands auto-resolve via the registry.

**`worker-cli response` is the default for reading idle workers; `capture` is the fallback.**
`response` gives clean assistant text from the session JSONL. `capture` is for when `response` misses context (rare — Phase-A partial-report situations); it returns the pane already cleaned and scoped to since the last prompt. Both print straight to context — never pipe them through `tail` / `head` / `sed`; a hook strips that anyway.

**Session name pattern.**
`worker-<basename(project_path)>-<name>`. Example: project `/Users/x/Monitor_CC` + worker `inject-fixes` → session `worker-Monitor_CC-inject-fixes`.

| Operation | CLI |
|---|---|
| List active workers (project) | `worker-cli list <project_path>` |
| List active workers (all) | `worker-cli list` |
| Check worker status | `worker-cli status <name>` |
| Clean output since last prompt | `worker-cli capture <name>` (`--raw` → raw pane to file) |
| Clean last N assistant turns (JSONL) | `worker-cli response <name> [N]` |
| Send message to running worker | `worker-cli send <name> <message>` |
| Merge worker branch | `worker-cli merge <name>` |
| Kill worker (+ registered cross-project worktrees) | `worker-cli kill <name>` |
| Spawn worker in worktree | `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` |
| Create cross-project worktree | `worker-cli worktree <name> <target_repo> [branch]` |
| Revive dead worker (resume CC session) | `worker-cli revive <name>` |

### Git CLI — Push, Merge & Orchestration

#### Push

| Operation | CLI | Notes |
|---|---|---|
| Push (NON-plugin repo) | `git -C <repo_path> push` | Falls back to `-u origin <branch>` if no upstream. **Use `plugin-publish` if `.claude-plugin/plugin.json` exists.** |
| Push with upstream (NON-plugin repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | For first push on new branch. |
| Push (PLUGIN repo) — replaces `git push` | `cd <plugin-source-repo> && plugin-publish` | One-step: git push + cache-sync + version-bump. **Always use this for any repo with `.claude-plugin/plugin.json`.** Never plain `git push` on a plugin repo. |

#### Commit Flow

When the user asks to commit:

1. **Commit** — `gcommit "<message>" [repo_path]`, mechanics in § Git CLI (Commit with `gcommit`). Want visibility first? Run `git-check [repo_path]` — it stages + prints a full STAGED/UNSTAGED/UNTRACKED + hook-status report WITHOUT committing (review-only, optional).
2. **Push** — does `<repo>/.claude-plugin/plugin.json` exist?
   - **YES (plugin repo):** `cd <repo> && plugin-publish` — git push + cache-sync + version-bump. NEVER `git push` here.
   - **NO (regular repo):** `git -C <repo> push` (retry with `-u origin <branch>` on first push).

#### Multi-Repo Commits

When committing multiple repos (e.g., project + plugin source):
- Run the full flow for each repo sequentially
- Plugin repos: use `plugin-publish` instead of `git push` (handles cache + restart automatically)

### RAG CLI

**RAG queries are ALWAYS written in English, regardless of conversation language.**
Issue the search command directly — no prior `rag-cli server start` needed.

**Indexed collections go through rag-cli; local source files through the Read tool.**
Indexed collections live in `data/documents/<collection>/`.

**`delete` removes three surfaces for the given scope: the matched chunks, their `indexed_files` manifest rows, and the on-disk source files under `data/documents/<collection>/`.**
`--collection` is required → deletes the whole collection (dir + all chunks + all manifest rows). `--document` (optional) narrows to one document → removes that doc's chunks + manifest row + its `.md` and `.json` sidecar. `--document` without `--collection` errors.

**`index` is the inverse of `delete` over the same scope: it chunks + embeds + stores `.md` files from `data/documents/<collection>/`.**
`--collection` is required → indexes every `.md` in the collection dir; `--document` (optional) → just that one file. Skip-by-default via content hash (unchanged files are skipped); `--force` re-embeds everything. `--document` without `--collection` errors.

**`search_hybrid` finds the hit; `read_document` pulls the context around it.**
`read_document <coll> <doc> <chunk> --before N --after M` returns that chunk plus its neighbors, where the useful detail usually sits.

**Every chunk you build on gets expanded with `read_document` first.**
The moment a chunk becomes the basis for a concrete action — writing an artifact like a rule or process-docs entry, but equally any action you take on the strength of it ("with knowledge X I do action Y") — expand it before you act; don't build on the bare search hit. A single chunk is a pointer, not the full context: what you need is often in the neighbors the search didn't return — the caveat sitting one chunk away that you'd otherwise miss.

**Current state comes from CODE, not RAG.**
The `<Project>-docs` collection indexes `process-docs/**` (write-once history) + `DOCS.md` (module map) — process-docs carries SUPERSEDED values that misread as current. So a "what IS the state of X" question is answered by reading the source code, not by a docs query. Use `<Project>-docs` when you want the reasoning / iteration history / why-it-was-decided; use the code (Read/grep) when you want the live value. DOCS.md is the one docs surface that tracks current code shape.

**Miss handling.**
On 0-chunk result, reformulate ≥ 2 phrasings before fallback to direct Read / bash `grep`. Partial hit short of answer: `read_document` with `--before N --after M` on the hit's `chunk_index`, not re-query.

| Operation | Command |
|---|---|
| List collections | `rag-cli list_collections [--filter PATTERN]` |
| List documents | `rag-cli list_documents <collection> [--document PATTERN] [--exclude PATTERN] [--filter PATTERN]` |
| Search hybrid | `rag-cli search_hybrid <query> <collection> [--document PATTERN] [--exclude PATTERN]` |
| Read context | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Delete | `rag-cli delete --collection <name> [--document <doc>]` |
| Index | `rag-cli index --collection <name> [--document <doc>]` |

### show — open a file for the user

**Open a file in the user's default macOS app so the USER can see it.**
Use it when the user asks to be shown a file: "öffne mir den Report" / "bring mir die py-Datei her und öffne sie" / "show me X" / "open the screenshot". The trigger is intent ("show ME"), not file type.

**Use `show` only when the user wants to LOOK at a file.**
For Claude-internal inspection (analysis, code review, grep) use the Read / Bash tools. Never use `show` for content Claude itself needs to consume.

| Operation | Command |
|---|---|
| Open one file | `show <path>` |
| Open multiple files | `show <p1> <p2> ...` |
| Relative path | `show ./report.md` |
| Home path | `show ~/Desktop/foo.png` |
