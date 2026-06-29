# Tool-Use — Orchestrator-Only

These tools are YOURS as orchestrator — workers never spawn workers, never query RAG, never push/merge.

## Worker CLI

Worker names are globally unique (registry-tracked). Project path is required only for `spawn`; other commands auto-resolve via the registry.

| Operation | CLI |
|---|---|
| List active workers (project) | `worker-cli list <project_path>` |
| List active workers (all) | `worker-cli list` |
| Check worker status | `worker-cli status <name>` |
| Clean output since last prompt | `worker-cli capture <name>` (`--raw` → raw pane to file) |
| Clean last N assistant turns (JSONL) | `worker-cli response <name> [N]` |
| Send message to running worker | `worker-cli send <name> <message>` |
| Merge worker branch | `worker-cli merge <name>` |
| Kill worker | `worker-cli kill <name>` |
| Spawn worker in worktree | `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` |
| Revive dead worker (resume CC session) | `worker-cli revive <name>` |

`worker-cli response <name>` is the default for reading idle workers — clean assistant text from the session JSONL. `worker-cli capture <name>` is the fallback when `response` misses context (rare — Phase-A partial-report situations); it returns the pane already cleaned and scoped to since the last prompt. Both print straight to context — never pipe them through `tail` / `head` / `sed`; a hook strips that anyway.

**Session name pattern:** `worker-<basename(project_path)>-<name>`. Example: project `/Users/x/Monitor_CC` + worker `inject-fixes` → session `worker-Monitor_CC-inject-fixes`.

**Examples:**

```bash
worker-cli list
worker-cli status inject-fixes
worker-cli response inject-fixes        # clean last assistant turn (default)
worker-cli capture inject-fixes         # clean pane since last prompt (fallback)
worker-cli merge inject-fixes
worker-cli kill inject-fixes   # only after status is idle/done
worker-cli send inject-fixes "Go for step 2"
worker-cli spawn new-feature /tmp/prompt.md c sonnet
```

## Git CLI — Push, Merge & Orchestration

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

| Operation | Command |
|---|---|
| List collections | `rag-cli list_collections [--filter PATTERN]` |
| List documents | `rag-cli list_documents <collection> [--document PATTERN] [--exclude PATTERN] [--filter PATTERN]` |
| Search hybrid | `rag-cli search_hybrid <query> <collection> [--document PATTERN] [--exclude PATTERN]` |
| Read context | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Delete | `rag-cli delete --collection <name> [--document <doc>]` |
| Index | `rag-cli index --collection <name> [--document <doc>]` |

### Rules

- RAG queries are ALWAYS written in English, regardless of conversation language.
- Issue the search command directly — no prior `rag-cli server start` needed.
- Indexed collections in `data/documents/<collection>/` → rag-cli. Local source files → Read tool, not rag-cli.
- `delete` removes three surfaces for the given scope: the matched chunks, their `indexed_files` manifest rows, and the on-disk source files under `data/documents/<collection>/`. `--collection` is required → deletes the whole collection (dir + all chunks + all manifest rows). `--document` (optional) narrows to one document → removes that doc's chunks + manifest row + its `.md` and `.json` sidecar. `--document` without `--collection` errors.
- `index` is the inverse of `delete` over the same scope: it chunks + embeds + stores `.md` files from `data/documents/<collection>/`. `--collection` is required → indexes every `.md` in the collection dir; `--document` (optional) → just that one file. Skip-by-default via content hash (unchanged files are skipped); `--force` re-embeds everything. `--document` without `--collection` errors.

### Search & expand

`search_hybrid` finds the hit; **`read_document <coll> <doc> <chunk> --before N --after M`** pulls that chunk plus the chunks around it, where the useful detail usually sits.

**Every chunk you build on gets expanded with `read_document` first.** The moment a chunk becomes the basis for an artifact — a rule, a decision, an IST description, anything you write — expand it before you use it; don't draft from the bare search hit. A single chunk is a pointer, not the full context: what you need is often in the neighbors the search didn't return — the caveat sitting one chunk away that you'd otherwise miss.

**Exclude process-memory from current-state queries.** `OldThemes/` nests under `decisions/` and carries SUPERSEDED values that misread as current. When the question is about the CURRENT state, append `--exclude "%OldThemes%"` to drop the whole OldThemes subtree (the `document` field is the full path, so one pattern catches it):

```bash
rag-cli search_hybrid "<query>" <Project>-docs --exclude "%OldThemes%"
```

Omit `--exclude` only when you specifically want iteration history / why-it-was-decided. Default for "what IS the state of X" = exclude OldThemes.

**Miss handling:** on 0-chunk result, reformulate ≥ 2 phrasings before fallback to direct Read / bash `grep`. Partial hit short of answer: `read_document` with `--before N --after M` on the hit's `chunk_index`, not re-query.

## show — open a file for the user

Open one or more files in the user's default macOS app so the **user** can see them. Use when the user asks to be shown a file: "öffne mir den Report" / "bring mir die py-Datei her und öffne sie" / "show me X" / "open the screenshot". The trigger is intent ("show ME"), not file type.

| Operation | Command |
|---|---|
| Open one file | `show <path>` |
| Open multiple files | `show <p1> <p2> ...` |
| Relative path | `show ./report.md` |
| Home path | `show ~/Desktop/foo.png` |

- **Use `show` only when the user wants to LOOK at a file.** For Claude-internal inspection (analysis, code review, grep) use the Read / Bash tools. Never use `show` for content Claude itself needs to consume.
