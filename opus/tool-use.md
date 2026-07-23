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

### Git

**Preview staged state before committing with `git-check [repo_path]`.**
Stages + prints a full STAGED/UNSTAGED/UNTRACKED + hook-status report WITHOUT committing — review-only, optional.

| Operation | CLI | Notes |
|---|---|---|
| Push (NON-plugin repo) | `git -C <repo_path> push` | Falls back to `-u origin <branch>` if no upstream. **Use `plugin-publish` if `.claude-plugin/plugin.json` exists.** |
| Push with upstream (NON-plugin repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | For first push on new branch. |
| Push (PLUGIN repo) — replaces `git push` | `cd <plugin-source-repo> && plugin-publish` | One-step: git push + cache-sync + version-bump. **Always use this for any repo with `.claude-plugin/plugin.json`.** Never plain `git push` on a plugin repo. |

### RAG CLI

**RAG queries are ALWAYS written in English, regardless of conversation language.**
Issue the search command directly — no prior `rag-cli server start` needed.

**`delete` removes three surfaces for the given scope: the matched chunks, their `indexed_files` manifest rows, and the on-disk source files under `data/documents/<collection>/`.**
`--collection` is required → deletes the whole collection (dir + all chunks + all manifest rows). `--document` (optional) narrows to one document → removes that doc's chunks + manifest row + its `.md` and `.json` sidecar. `--document` without `--collection` errors.

**`index` is the inverse of `delete` over the same scope: it chunks + embeds + stores `.md` files from `data/documents/<collection>/`.**
`--collection` is required → indexes every `.md` in the collection dir; `--document` (optional) → just that one file. Skip-by-default via content hash (unchanged files are skipped); `--force` re-embeds everything. `--document` without `--collection` errors.

**`search` finds the hit; `read_document` pulls the context around it.**
`read_document <coll> <doc> <chunk> --before N --after M` returns that chunk plus its neighbors, where the useful detail usually sits.

**Every chunk you build on gets expanded with `read_document` first — NON-NEGOTIABLE.**
Before ANY action rests on a chunk — writing an artifact (rule / process-docs / DOCS / code), stating a grounded claim, or any "with knowledge X → do Y" — first run `read_document <coll> <doc> <chunk> --before N --after M` on that chunk. Every chunk you build on, each one, zero exceptions. A bare `search` hit is NEVER a sufficient basis to act on. Believing you already have all the info you need is irrelevant and is NEVER grounds to skip: the expansion is mandatory regardless of that belief. Build on N chunks → N expansions before you act.

**Miss handling.**
On 0-chunk result, reformulate ≥ 2 phrasings before fallback to direct Read / bash `grep`. Partial hit short of answer: `read_document` with `--before N --after M` on the hit's `chunk_index`, not re-query.

| Operation | Command |
|---|---|
| List collections | `rag-cli list_collections [--filter PATTERN]` |
| List documents | `rag-cli list_documents <collection> [--document PATTERN] [--exclude PATTERN] [--filter PATTERN]` |
| Search | `rag-cli search <query> <collection> [--document PATTERN] [--exclude PATTERN]` |
| Read context | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Delete | `rag-cli delete --collection <name> [--document <doc>]` |
| Index | `rag-cli index --collection <name> [--document <doc>]` |

### GitHub Issues (gh-cli) — Cross-Session Context

**Derive `<owner>` and `<repo>` from the git remote; never hardcode.**
`git remote get-url origin` returns `github.com:<owner>/<repo>.git` — pull both from there.

**Default = open only.**
`gh-cli list_issues` shows OPEN issues by default; closed appear only with `--state closed` (or `--state all`). Pull requests are filtered out.

**Finding the number.**
`gh-cli list_issues <owner> <repo>` lists open issues one per line (`#N [OPEN] title`); match by title and pass the stable `<number>`.

#### What an Issue IS

An Issue is a **lean entry-point**: topic + sources that reference it. Content lives elsewhere:

- source code — the current architectural state (read the code; there is no doc mirror)
- `process-docs/<area>/` — process history: investigation, measurements, iteration, the reasoning behind chosen code values (write-once entries)
- `<package>/DOCS.md` — module map
- RAG `<Project>-reference` collection — external sources (vendor docs, papers, GitHub, Reddit, repos)

Resume mechanism: RAG-search on `<Project>-docs` (DOCS/CLAUDE/process-docs) + reading the code, `<Project>-reference` (external sources).

Issues are created at exactly two points: when the user asks mid-session, or at Recap for whatever is still open at session end. There is no autonomous mid-session issue-keeping between those.

#### Issue Format

The issue body carries the entry-point. Title = the feature/bug/task name; body:

```
What it is:
[2-3 sentences — goal + scope. No iteration history. No decision rationale.]

Sources referencing this topic:
- code: <key src/ paths if any>
- DOCS: <DOCS.md paths if any>
- process-docs: <subfolder or file paths if any>
- <Project>-reference: <document names if any>

Resume: RAG search "<query>" on <Project>-docs
```

Source paths relative to project root. The Source-Inventory is a snapshot at the moment of writing — Recap is responsible for keeping it current.

#### Resume Pattern

When picking up an open issue in a new session:

1. Read the issue: `gh-cli get_issue <owner> <repo> <number>` — the body carries the Source-Inventory (no comments to read)
2. RAG search for context:
   - `<Project>-docs` — current state + discussion trail / iteration history
   - `<Project>-reference` — external sources (vendor docs, papers, repos)

The issue does not contain narrative. The sources do.

#### Issue-Close

`gh-cli update_issue <owner> <repo> <number> --state closed` — that's it.

Close proactively: when the issue's code is merged AND live-verify shows the new behavior works as intended, close it in the same flow — don't wait for the user to ask.

No verification of prosa-state at close (Recap is responsible for persistence). The process-docs prosa is the journey summary — nothing is posted to the issue.

If an issue defines a specific verification test that has not been run yet → issue stays open, run the test, then close.

| Operation | CLI |
|---|---|
| List open issues | `gh-cli list_issues <owner> <repo>` (state=open is the default) |
| List closed issues | `gh-cli list_issues <owner> <repo> --state closed` |
| Read issue body | `gh-cli get_issue <owner> <repo> <number>` — body = text AFTER the `---` separator in the output |
| Create issue | `gh-cli create_issue <owner> <repo> "<title>" --body "<desc>" [--labels a,b]` |
| Update issue body (Source-Inventory) | `gh-cli update_issue <owner> <repo> <number> --body "<full updated body>"` (full-replace) |
| Close issue | `gh-cli update_issue <owner> <repo> <number> --state closed` |
| Reopen issue | `gh-cli update_issue <owner> <repo> <number> --state open` |

### show — open a file for the user

**Open a file in the user's default macOS app so the USER can see it.**
Use it when the user asks to be shown a file: "öffne mir den Report" / "bring mir die py-Datei her und öffne sie" / "show me X" / "open the screenshot". The trigger is intent ("show ME"), not file type.

**Use `show` only when the user wants to LOOK at a file.**
For Claude-internal inspection (analysis, code review, grep) use the Read / Bash tools. Never use `show` for content Claude itself needs to consume.

**A file already opened with `show` stays open — don't re-`show` it after each edit.**
One `show` at first display holds for the whole session; the open app picks up later edits in place, so re-running it per edit is redundant.

| Operation | Command |
|---|---|
| Open one file | `show <path>` |
| Open multiple files | `show <p1> <p2> ...` |
| Relative path | `show ./report.md` |
| Home path | `show ~/Desktop/foo.png` |
