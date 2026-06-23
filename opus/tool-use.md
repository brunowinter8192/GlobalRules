# Tool-Use — Orchestrator-Only

Worker-lifecycle commands, RAG search, and Git push/merge orchestration. These tools are YOURS as orchestrator — workers never spawn workers, never query RAG, never push/merge. The shared tool reference (Bash, commit primitives, Read/Edit/Write, grep hygiene) lives in `global/tool-use.md`, which YOU also receive.

## Worker CLI

Worker names are globally unique (registry-tracked). Project path is required only for `spawn`; other commands auto-resolve via the registry.

All worker lifecycle operations via `~/.local/bin/worker-cli`.

| Operation | CLI |
|---|---|
| List active workers (project) | `worker-cli list <project_path>` |
| List active workers (all) | `worker-cli list` |
| Check worker status | `worker-cli status <name>` |
| Capture pane to file | `worker-cli capture <name>` |
| Read last N lines | `tail -n <N> <output_file_from_capture>` |
| Get clean last response | `worker-cli response <name>` |
| Send message to running worker | `worker-cli send <name> <message>` |
| Merge worker branch | `worker-cli merge <name>` |
| Kill worker | `worker-cli kill <name>` |
| Spawn worker in worktree | `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` |
| Revive dead worker (resume CC session) | `worker-cli revive <name>` |

`worker-cli response <name>` is the default for reading idle workers — returns clean text from session JSONL (~200-2000 chars, no UI trailers or prompt echo). `worker-cli capture <name>` + `tail` + `sed`-filter is the fallback when `response` misses context (rare — Phase-A partial-report situations). Capture dumps 2-5k chars of CC UI + prompt echo.

Polling flow (timer → status → response, with flow-level discipline like "one timer at a time", foreground vs background, etc.) is orchestration, not a command-level concern — see `opus/workers-2.md` "Timer & Polling Flow".

The wrapper internally sources `$PLUGIN/src/spawn/tmux_spawn.sh`. Override plugin location via `CLAUDE_PLUGIN_ROOT` env var.

**Session name pattern:** `worker-<basename(project_path)>-<name>`. Example: project `/Users/x/Monitor_CC` + worker `inject-fixes` → session `worker-Monitor_CC-inject-fixes`.

**Fallback** (wrapper unavailable):

```bash
PLUGIN=~/.claude/plugins/cache/brunowinter-plugins/iterative-dev/1.0.0
SPAWN="$PLUGIN/src/spawn/tmux_spawn.sh"
bash -c "source \"$SPAWN\" && worker_status \"<name>\" \"<project_path>\""
```

**Examples:**

```bash
worker-cli list
worker-cli status inject-fixes
worker-cli capture inject-fixes
# → prints path like /tmp/worker_capture_inject-fixes_123456.txt
tail -n 50 /tmp/worker_capture_inject-fixes_123456.txt
worker-cli merge inject-fixes
worker-cli kill inject-fixes   # only after status is idle/done
worker-cli send inject-fixes "Go for step 2"
worker-cli spawn new-feature /tmp/prompt.md c sonnet
```

## Git CLI — Push, Merge & Orchestration

Commit primitives (`git-check`, `gc`, commit message format, safety protocol) → `global/tool-use.md` § Git CLI, which YOU also receive. This section covers push, the end-to-end commit→push flow, and multi-repo orchestration — none of which workers perform.

### Push

| Operation | CLI | Notes |
|---|---|---|
| Push (NON-plugin repo) | `git -C <repo_path> push` | Falls back to `-u origin <branch>` if no upstream. **Use `plugin-publish` if `.claude-plugin/plugin.json` exists.** |
| Push with upstream (NON-plugin repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | For first push on new branch. |
| Push (PLUGIN repo) — replaces `git push` | `cd <plugin-source-repo> && plugin-publish` | One-step: git push + cache-sync + version-bump. **Always use this for any repo with `.claude-plugin/plugin.json`.** Never plain `git push` on a plugin repo. |

### Commit Flow

When user asks to commit:

1. **Check + Stage** — `git-check [repo_path]`
2. **Commit** — `gc "<message>"` (if cwd inside repo) OR `git -C <repo> commit -am "<message>"` (explicit path)
3. **Post-check** — `git -C <repo> status --short` → empty = proceed; non-empty → stage + commit again
4. **Push** — first check: does `<repo>/.claude-plugin/plugin.json` exist?
   - **YES (plugin repo):** `cd <repo> && plugin-publish` — does git push + cache-sync + version-bump. NEVER `git push` here.
   - **NO (regular repo):** `git -C <repo> push` (retry with `-u origin <branch>` on first push).

### Multi-Repo Commits

When committing multiple repos (e.g., project + plugin source):
- Run the full flow for each repo sequentially
- Plugin repos: use `plugin-publish` instead of `git push` (handles cache + restart automatically)

## RAG CLI

Indexed-document search and lookup. All RAG operations via `rag-cli` (`~/.local/bin/rag-cli`).

| Operation | Command |
|---|---|
| List collections | `rag-cli list_collections [--filter PATTERN]` |
| List documents | `rag-cli list_documents <collection> [--document PATTERN] [--filter PATTERN]` |
| Search hybrid | `rag-cli search_hybrid <query> <collection> [--document PATTERN]` |
| Read context | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Delete | `rag-cli delete --collection <name> [--document <doc>]` |
| Index | `rag-cli index --collection <name> [--document <doc>]` |

### Rules

- Issue the search command directly — no prior `rag-cli server start` needed.
- On persisted-output: read the file completely in ONE Read call, no offset/limit chunking.
- Indexed collections in `data/documents/<collection>/` → rag-cli. Local source files → Read tool, not rag-cli.
- `delete` removes three surfaces for the given scope: the matched chunks, their `indexed_files` manifest rows, and the on-disk source files under `data/documents/<collection>/`. `--collection` is required → deletes the whole collection (dir + all chunks + all manifest rows). `--document` (optional) narrows to one document → removes that doc's chunks + manifest row + its `.md` and `.json` sidecar. `--document` without `--collection` errors.
- `index` is the inverse of `delete` over the same scope: it chunks + embeds + stores `.md` files from `data/documents/<collection>/`. `--collection` is required → indexes every `.md` in the collection dir; `--document` (optional) → just that one file. Skip-by-default via content hash (unchanged files are skipped); `--force` re-embeds everything. `--document` without `--collection` errors.

### RAG: Status-Quo via RAG first

Trigger: project has `.rag-docs.json` at root → `<Project>-docs` collection exists with decisions/, DOCS.md, OldThemes indexed.

**Status-quo questions answered by RAG, not by direct-read of decisions/:**
- "What is the IST of X?" / "How does Y work?" / "What was decided about Z?"

```bash
rag-cli search_hybrid "<query>" <Project>-docs
```

For a simple status lookup the chunk is the answer. For important chunks, expand via `read_document`, not a direct-read of the file.

**Direct-read on the full decision file ONLY when:**
- The file is being EDITED (need all sections in view)
- The file was edited THIS session and RAG hasn't been resynced
- RAG returned no usable hit AND the path is known anyway
- The answer needs more context → expand via `rag-cli read_document <coll> <doc> <chunk> --after N`, NOT raw direct-read

**Search commands by use case:**

| Use case | Command |
|---|---|
| Content search | `rag-cli search_hybrid <query> <coll>` |
| Expand context around a hit | `rag-cli read_document <coll> <doc> <chunk> --before N --after M` |

**RAG search (the pattern other rules reference):** `search_hybrid` first; for the most important chunks also `read_document <coll> <doc> <chunk_index> --before N --after M`. "Use RAG search" elsewhere means exactly this.

Defaults: `top_k` is hardcoded to 10 in `search_hybrid_workflow` (not configurable, no flag exposed). Reranking is always on — `search_hybrid` unconditionally cross-encoder-reranks the top-30 dense candidates; there is no `--rerank` flag to toggle. `--document` filter narrows to matching doc names (optional).

**Two collection layers per project** — `<Project>-docs` (internal) + `<Project>-reference` (external). Full convention: `global/documentation.md` § RAG Collection Layers. Reference is on-demand only, not part of the routine docs query.

**Miss handling:** on 0-chunk result, reformulate ≥ 2 phrasings before fallback to direct Read / bash `grep`. Partial hit short of answer: `read_document` with `--before N --after M` on the hit's `chunk_index`, not re-query.
