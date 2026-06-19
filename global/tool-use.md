# Tool-Use

Goal: no large Bash-call where a small one works.

## Hard Rules — Token Efficiency

### 1. Python: heredoc for one-shot, Write + Edit for iteration (binary rule)

**Decision tree:**

1. **jq / grep / awk first** if the data shape fits — one-liners beat Python.
2. **Python heredoc** when Python is needed AND the script runs once.
3. **Write + Edit** ONLY when the SAME script will run a second time after edits.

Different script for a different question is a NEW one-shot — five probes answering five questions = five heredocs, not five Writes.

**`python3 -c`:** switch to heredoc when the `-c` string exceeds ~300 chars including escaping.


### 3. Grep scope hygiene — always restrict when searching source

- When searching for Python imports, function refs, or code-level patterns: pass `--include='*.py'` to bash `grep` to scope to source files.
- For a recursive search across a tree, always pass an explicit file glob: `grep -rn --include='<glob>' <pattern> <dir>` — never a bare `-r` over the whole tree.
- For one-off bash grep: add explicit file scope (`grep -n <pattern> <specific_file>`) rather than `-r` over a whole tree.

Hook `block_broad_grep.py` enforces.


### 4. Context window hygiene — verbose output to file, not context

Noisy outputs go through files; signal outputs go directly into context.

**File-redirect path** (most output is noise, you'll grep for the signal):
- Build output, test runners, dev scripts with debug logs, background workflows

```bash
./venv/bin/python dev/crawling_suite/03_test.py > /tmp/03_test_output.md 2>&1
tail -20 /tmp/03_test_output.md
```

**Direct-to-context path** (the output IS the answer):
- Search-CLI / RAG tool results
- Single-file reads via `cat` / `head` / `tail` of bounded size
- Git status / log / diff (when bounded)

```bash
rag-cli search_hybrid "query" <Project>-docs
```

- NEVER run `./venv/bin/python script.py` without `> /tmp/file.md 2>&1`
- NEVER run `cat` on a file that might be large — use `head`, `tail`, `grep`

### 5. Stop after 2 failed tool calls

When 2 tool calls in a row fail or don't deliver the desired result: **STOP IMMEDIATELY**.

- Clearly explain the problem to the user
- Ask: "How should I solve this?" or "Where can I find X?"
- NO further trial and error without user input
- After 2 failures: the answer is RESEARCH (Web/GitHub search, read source code, read docs), not RETRY


### 6. One Bash tool_use block per response — chain inside it

**Correct dispatch pattern:** each assistant response carries at most ONE Bash tool_use block. Put every Bash-class action for that turn INSIDE that single block, chained with `;` or `&&` — no limit on command count.

- Independent commands → chain with `;`
- Dependent commands (each step prerequisite for the next) → chain with `&&`
- Diagnostic chains where step failure must NOT abort the rest → `;` (see Rule 11)

Sequential-across-turns is the right move only when a LATER command needs the OUTPUT of an EARLIER one.

Applies to ALL Bash invocations. Read/Write/Edit may be sequenced together in one response when there is a genuine ordering need (e.g. Read→Edit per Rule 9); a Bash block always travels alone.

**Chain everything chainable.** When dispatching a Bash call, identify what other Bash-class actions are the obvious next step and pack them into the same block:

- After `git merge`: chain post-merge verification (`; rag-cli search_hybrid "test" rag-cli-docs`)
- After `worker-cli send X`: chain status check of other workers (`; worker-cli list`)
- After identifying a bug via investigation: chain the fix-dispatch (`; worker-cli send X "fix Y"`)
- Cleanup actions (`rm /tmp/scratch.*`, `git stash drop`) cost zero extra tool calls when chained — always chain them.

**Verbal-deferral is forbidden.** Phrases like "I'll do X next turn" / "Timer setze ich gleich" / "ich verifiziere das anschließend" / "Mache ich später" trigger an immediate self-check: **could X have been chained into the current Bash call?**

- If YES → rule violation. Rewrite the response, chain the action.
- If NO (genuine tool-constraint: background-Bash + foreground-Bash conflict, Read-required-before-Edit that did not fit, etc.) → state the explicit constraint AND specify the exact next-turn first-action as a concrete command, not a vague promise.

Investigation, follow-up, verification all count as chainable same-turn actions: `worker-cli response`/`status`, `grep` on an identified pattern, `rag-cli` verify, post-merge tests, diff review, issue comment, file delete. These are NEVER next-turn material when current Bash budget is available.


### 7. Tool failure → immediate action

Tool call fails → do NOT continue with workaround or fallback without reporting.

1. Tool fails → report error to user in same message
2. Can fix it (start process, install dep, create dir)? → fix NOW, retry
3. Can't fix it (needs user credentials, hardware, manual step)? → stop, explain what's needed
4. NEVER silently switch to plan B without disclosing plan A failed
5. NEVER ask "should I start X or work around it?" — if you CAN fix it, fix it


### 8. `<persisted-output>` blocks: ALWAYS Read the full file — never grep, never preview

When a tool_result contains a `<persisted-output>` block (`Full output saved to: <path>`), ALWAYS Read the full file via the Read tool — one Read call on the path. NEVER grep it, NEVER head/tail/cat-filter it, NEVER offset/limit-chunk it, NEVER settle for the `Preview (first NKB)` content.

**Workflow:**
1. Extract the absolute path from `Full output saved to: <path>`.
2. Read the ENTIRE file in ONE Read call. If it exceeds 2000 lines, raise `limit=N` to cover it.

**Only exception — physical Read-tool limits:** the Read tool hard-fails on files >256KB or >25k tokens. ONLY in that case fall back to a targeted `grep -n <target> <path>` to find the line, then a bounded Read with `offset`/`limit` around it. Typical persisted tool/RAG outputs are tens of KB — this exception almost never fires. When in doubt, Read the whole file.


### 9. Read before Edit/Write — non-negotiable

Before any Edit or Write call on an existing file, call Read on the same path. ONE Read per file per session is enough — subsequent Edit/Write calls reuse the read state. Read followed by Edit in the same response is fine; both fire in order.

**Forbidden shortcut:** "I know the content already" or "I just edited this in another session" does NOT satisfy. Read-state is per-session, not persisted across sessions.


### 10. gh-cli reads: full output into context — never pipe through a filter

gh-cli output is signal, not noise — it IS the file content / search result you asked for. Take it WHOLE into context. NEVER pipe a gh-cli call through `grep` / `head` / `tail` / `sed` / `awk` or any shell filter, and never chain a gh-cli read into one.

- **Forbidden:** `gh-cli get_file_content … | grep X`, `gh-cli search_code … | head`, `gh-cli … | sed -n …`. Filtering a gh-cli read means Opus only ever sees fragments and tries to reconstruct the whole from them — which costs MORE context than the full read would have, and usually misses the answer anyway.
- **To narrow scope, use gh-cli's OWN flags**, never a downstream shell filter: `get_file_content --offset N --limit M` reads a bounded file region, `--metadata-only` returns size/shape, `search_code`'s query + `repo:`/`path:` qualifiers narrow the hit set, `--document`/`--filter` narrow listings. Then take that bounded output in full.
- **Standalone, but chainable with other gh-cli reads.** A gh-cli call runs on its own — no filter attached. Multiple gh-cli READ calls may sit back-to-back in one Bash block (e.g. `gh-cli get_file_content A; gh-cli get_file_content B`); what breaks is mixing a gh-cli read with a filter/pipe in the same pipeline.

Same principle as §8 (`<persisted-output>`: read the full file, never grep). If a gh-cli read is genuinely too large for one pass, narrow it with the tool's `--offset`/`--limit`, not with a downstream `grep`.


### 11. Diagnostic Bash chains: `;` not `&&`

**Hard test before writing `cmd_a && cmd_b`:** ask whether `cmd_b` would exit non-zero in a normal correct case — e.g. `grep` no-match, `ls` on empty/missing directory, `test -f` on a path that may not exist, `wc` on missing file. If yes → use `;`.

`&&` is reserved for sequences where step N's success is a *prerequisite* for step N+1: build before test, mkdir before write, commit before push, install before run. Diagnostic chains, verification probes, multi-source sanity-checks always use `;`.

```bash
# WRONG — grep no-match aborts the rest
echo "=== refs ===" && grep X file && ls dir/ && echo "=== done ==="

# RIGHT — each step runs regardless
echo "=== refs ==="; grep X file; ls dir/; echo "=== done ==="
```

`2>/dev/null` swallows stderr but does NOT change exit codes — adding it does not save the chain.

**Exit code = last command.** If the last step is a conditional-then-action (`[ ... ] && X`, `grep X && Y`), fix it:
- `[ -f path ] && tail path || true` — force exit 0
- `if [ -f path ]; then tail path; fi` — returns 0 when path absent
- Append `; true` to the whole chain to guarantee exit 0


### 13. Worktree path is `.claude/worktrees/` — never `.claire/`

The worktree directory is `.claude/worktrees/<name>/`. There is no `.claire/` anywhere.

**Rule:** before any file operation naming a worktree path explicitly, verify the literal substring `.claude/worktrees/` (with `u` and `d`, not `i` and `r`). When cwd is inside a worktree, prefer relative paths or the `c` shortcut for `worker-cli` rather than reconstructing absolute paths.

**Same-class typo:** `..letter` (two consecutive dots immediately followed by lowercase letter, e.g. `..claude/`, `..src/`) is virtually never a valid path component. Valid parent-traversal is `../` (two dots + slash).

Hook `block_path_typo.py` enforces both patterns.


### 14. Background Bash is a deliberate choice, never a default

`run_in_background=true` MUST be a conscious decision — never typed by default for grep / cat / ls / read-only commands.

**Use background ONLY for:**
- Long-running commands the user explicitly wants to overlap with other work (build/test runs >30s, `sleep N && echo done` orchestration timer)
- Worker-spawn timer patterns (opus-workers-2)

**Never use background for:**
- Quick grep / cat / ls / wc / git status / file inspection
- Anything you'll read the output of within the same response


### 15. zsh Quoting for Repeated Path Calls

When a command is called repeatedly with the same long paths, NEVER pack two paths into a single variable with whitespace. Three robust patterns:

1. **Function:** `cmd() { /full/python /full/cli.py "$@"; }`
2. **Two-Variable-Split:** `PY=/full/python; CLI=/full/cli.py; $PY $CLI ...`
3. **Wrapper script in `~/.local/bin/`:** 3-line bash wrapper + `chmod +x`, then just `cmd-name ...` (pattern: rag-cli, reddit-cli, arxiv-cli)


### 16. cd-Drift across Bash-Tool-Calls

**Rule:** when a Bash call contains `cd "$WORKTREE"` or similar, the last step MUST cd back to the main cwd. Alternative: use `git -C <path>` and absolute paths throughout, never cd-ing.


---

## Soft Rules

### Repeated absolute paths → env var or single `cd`

When a path appears in 3+ consecutive commands: use `$PROJECT_ROOT` (or equivalent) or `cd` once at the start of a planned block. Do NOT drift `cd` silently across interactive steps — only within a contained sequence.

### Don't chain greps/cats over the same pattern

```
WRONG: grep X a.log && grep X b.log && grep X c.log
RIGHT: grep X a.log b.log c.log
```

### Don't re-issue near-identical commands

If you've run a command in this session and the output wasn't what you needed, do NOT retry with a minor variation. Change approach: a different scope, locate files first with `find`/`ls`, or read source directly.

### grep gunshot

Multiple bash `grep` calls with varied patterns that all return zero results = guessing. Pattern:
1. Locate candidate files first with `find`/`ls` (broad path pattern).
2. Targeted `grep -n <pattern>` on the located file.

Two zero-results in a row on the same topic = stop, rethink.

---

## Large Artifacts

### Heredoc — three cases

**Case 1 — Python / analysis:** one-shot = heredoc, iteration (run again with changes) = Write + Edit. See Rule 1.

**Case 2 — File creation or editing:** NEVER Bash heredoc / `cat > file <<EOF` / `tee file <<EOF`. Always Write (new file) or Edit (existing file). This includes `.py`, `.sh`, `.md`, config files, scripts — any file living in the repo. The only files you may write via heredoc are throwaways under `/tmp/`.

**Case 3 — Shell-argument heredoc for a one-shot command (git commit body, curl payload):** OK.

```bash
curl -X POST https://api.example.com/resource -d "$(cat <<'EOF'
{"field": "multi-line\nvalue"}
EOF
)"
```

**Decision flow:**

1. Python / analysis code? → jq/awk/sed first. If Python needed: one-shot → heredoc. Will rerun with changes → Write + Edit from run #2.
2. Goal is to create a file? → Write tool.
3. Multi-line argument to a one-shot shell command? → heredoc inline.
4. Same multi-line content reused across multiple calls? → Write + Edit.

---

## Tool-Specific Reference

### Bash
- **Timeout:** default 120000ms (2 min), max 600000ms (10 min). Use the `timeout` parameter when running long builds, crawls, or test suites to avoid silent termination.

#### Worker CLI

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

Polling flow (timer → status → response, with flow-level discipline like "one timer at a time", foreground vs background, etc.) is orchestration, not a command-level concern — see opus-workers-2 "Timer & Polling Flow".

The wrapper internally sources `$PLUGIN/src/spawn/tmux_spawn.sh`. Override plugin location via `CLAUDE_PLUGIN_ROOT` env var.

**Session name pattern:** `worker-<basename(project_path)>-<name>`. Example: project `/Users/x/Monitor_CC` + worker `inject-fixes` → session `worker-Monitor_CC-inject-fixes`.

**NEVER kill without checking status first.** If status is `working` → do NOT kill.

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

#### Git CLI

Pre-commit check via `git-check` CLI, everything else via CLI.

##### Pre-Commit

`git-check [repo_path]` — `repo_path` accepts `c` (same resolver logic as worker-cli). Auto-stages files (with skip patterns: venv/, node_modules/) and returns a status report:
- `STAGED` / `UNSTAGED` / `UNTRACKED` sections
- `HOOK STATUS`
- `DIFF SUMMARY` → use for commit message

If all sections are `(none)` → nothing to commit, skip.

##### CLI Commands

| Operation | CLI | Notes |
|---|---|---|
| Commit (inside repo/worktree cwd) | `gc "<message>"` | Wrapper: stages tracked modifications + commits. Add filenames as extra args to stage specific files. |
| Commit (explicit path) | `git -C <repo_path> commit -am "<message>"` | Use when cwd is outside target repo. `-am` stages tracked mods. For untracked: `git -C <path> add <files> && git -C <path> commit -m "<msg>"`. |
| Push (NON-plugin repo) | `git -C <repo_path> push` | Falls back to `-u origin <branch>` if no upstream. **Use `plugin-publish` if `.claude-plugin/plugin.json` exists.** |
| Push with upstream (NON-plugin repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | For first push on new branch. |
| Post-commit check | `git -C <repo_path> status --short` | Empty output = clean working tree. |
| Push (PLUGIN repo) — replaces `git push` | `cd <plugin-source-repo> && plugin-publish` | One-step: git push + cache-sync + version-bump. **Always use this for any repo with `.claude-plugin/plugin.json`.** Never plain `git push` on a plugin repo. |

##### Commit Flow

When user asks to commit:

1. **Check + Stage** — `git-check [repo_path]`
2. **Commit** — `gc "<message>"` (if cwd inside repo) OR `git -C <repo> commit -am "<message>"` (explicit path)
3. **Post-check** — `git -C <repo> status --short` → empty = proceed; non-empty → stage + commit again
4. **Push** — first check: does `<repo>/.claude-plugin/plugin.json` exist?
   - **YES (plugin repo):** `cd <repo> && plugin-publish` — does git push + cache-sync + version-bump. NEVER `git push` here.
   - **NO (regular repo):** `git -C <repo> push` (retry with `-u origin <branch>` on first push).

##### Commit Message Format

**Default: single-line `-m`, one concern per commit, ≤72 chars.**

```bash
gc "fix: reset warnings pane on proxy log path change"
# or
git -C <repo> commit -am "fix: reset warnings pane on proxy log path change"
```

- Types: `feat` / `fix` / `refactor` / `docs` / `chore`
- Max 72 chars
- Pick dominant concern if mixed
- No Co-Author footer for routine commits

**Multi-line HEREDOC ONLY when ALL are true:**
- Breaking change, migration, or architecturally significant refactor
- Body genuinely adds information beyond the subject line
- The reader of `git log` will benefit from the extra context

```bash
git -C <repo> commit -am "$(cat <<'EOF'
refactor: migrate X from Y to Z

Breaking: consumers of Y must update to new signature (see MIGRATION.md).
EOF
)"
```

HEREDOC for routine fixes is waste. Single-line `-m` is the default.

##### Multi-Repo Commits

When committing multiple repos (e.g., project + plugin source):
- Run the full flow for each repo sequentially
- Plugin repos: use `plugin-publish` instead of `git push` (handles cache + restart automatically)

##### Rules (Safety Protocol)

- NEVER amend existing commits
- NEVER force push
- NEVER skip hooks (`--no-verify`)
- NEVER modify git config
- NEVER create empty commits
- If push fails → report error, do NOT retry
- Commit only when user explicitly asks

#### RAG CLI

Indexed-document search and lookup. All RAG operations via `rag-cli` (`~/.local/bin/rag-cli`).

| Operation | Command |
|---|---|
| List collections | `rag-cli list_collections [--filter PATTERN]` |
| List documents | `rag-cli list_documents <collection> [--document PATTERN] [--filter PATTERN]` |
| Search hybrid | `rag-cli search_hybrid <query> <collection> [--document PATTERN]` |
| Read context | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Delete | `rag-cli delete --collection <name> [--document <doc>]` |
| Index | `rag-cli index --collection <name> [--document <doc>]` |

##### Rules

- Issue the search command directly — no prior `rag-cli server start` needed.
- On persisted-output: read the file completely in ONE Read call, no offset/limit chunking.
- Indexed collections in `data/documents/<collection>/` → rag-cli. Local source files → Read tool, not rag-cli.
- `delete` removes three surfaces for the given scope: the matched chunks, their `indexed_files` manifest rows, and the on-disk source files under `data/documents/<collection>/`. `--collection` is required → deletes the whole collection (dir + all chunks + all manifest rows). `--document` (optional) narrows to one document → removes that doc's chunks + manifest row + its `.md` and `.json` sidecar. `--document` without `--collection` errors.
- `index` is the inverse of `delete` over the same scope: it chunks + embeds + stores `.md` files from `data/documents/<collection>/`. `--collection` is required → indexes every `.md` in the collection dir; `--document` (optional) → just that one file. Skip-by-default via content hash (unchanged files are skipped); `--force` re-embeds everything. `--document` without `--collection` errors.

##### RAG: Status-Quo via RAG first

Trigger: project has `.rag-docs.json` at root → `<Project>-docs` collection exists with decisions/, DOCS.md, OldThemes indexed.

**Status-quo questions answered by RAG, not by direct-read of decisions/:**
- "What is the IST of X?" / "How does Y work?" / "What was decided about Z?"

```bash
rag-cli search_hybrid "<query>" <Project>-docs
```

The returned chunk IS the answer. No follow-up direct-read of the same file needed.

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

Defaults: `top_k` is hardcoded to 10 in `search_hybrid_workflow` (not configurable, no flag exposed). Reranking is always on — `search_hybrid` unconditionally cross-encoder-reranks the top-30 dense candidates; there is no `--rerank` flag to toggle. `--document` filter narrows to matching doc names (optional).

**Two collection layers per project** — `<Project>-docs` (internal) + `<Project>-reference` (external). Full convention: `~/.claude/shared-rules/global/documentation.md` § RAG Collection Layers. Reference is on-demand only, not part of the routine docs query.

**Miss handling:** on 0-chunk result, reformulate ≥ 2 phrasings before fallback to direct Read / bash `grep`. Partial hit short of answer: `read_document` with `--before N --after M` on the hit's `chunk_index`, not re-query.

#### show — open a file for the user

Open one or more files in the user's default macOS app so the **user** can see them. Use when the user asks to be shown a file: "öffne mir den Report" / "bring mir die py-Datei her und öffne sie" / "show me X" / "open the screenshot". The trigger is intent ("show ME"), not file type.

| Operation | Command |
|---|---|
| Open one file | `show <path>` |
| Open multiple files | `show <p1> <p2> ...` |
| Relative path | `show ./report.md` |
| Home path | `show ~/Desktop/foo.png` |

- **Use `show` only when the user wants to LOOK at a file.** For Claude-internal inspection (analysis, code review, grep) use the Read / Bash tools. Never use `show` for content Claude itself needs to consume.
- macOS picks the app: Preview for images/PDF, default editor for code/markdown, etc.
- Relative paths resolve against current pwd; `~` expanded.
- Errors with exit 1 if any path is missing — fix the path and retry; do NOT swallow the error.

### Read
- **Line limit:** reads up to 2000 lines by default. Use `offset` + `limit` parameters for larger files.
- **Output format:** `cat -n` format — `line_number\tcontent`.
- **Images:** can read PNG, JPG, etc. — presented visually as a multimodal model.
- **PDF:** files with >10 pages MUST include a `pages` parameter (e.g. `"1-5"`). Omitting it on large PDFs causes a tool failure. Max 20 pages per request.
- **Jupyter:** can read `.ipynb` notebooks — returns all cells with their outputs.
- **Directories:** Read cannot read directories. Use `ls` via Bash.
- **Empty file:** returns a system-reminder warning in place of content — do NOT interpret the warning as actual file content.
- **256KB limit:** files >256KB fail with `File content (Xkb) exceeds maximum allowed size (256KB)`. Pre-check with `wc -c <file>` or `ls -la` for large logs/JSONL. Fix: `grep -n <target> <file>` to find line, then `Read(file_path=..., offset=N, limit=200)`.
- **25k-token limit:** files >25k tokens fail with `File content (X tokens) exceeds maximum allowed tokens (25000)`. Same fix: grep + targeted Read with offset/limit.
- **Nonexistent path:** fails with `File does not exist. Note: your current working directory is …`. Verify path with `ls` before Read when path is reconstructed from memory. Common typos: `.claire/` (should be `.claude/`), `..claude/` (double-dot — never valid).
### Edit
- **Read first:** Required — see Rule 9.
- **Indentation:** preserve EXACT indentation as it appears in the file content.
- **Uniqueness:** FAIL if `old_string` is not unique in the file. Remedy: expand the match string with more surrounding context, or use `replace_all`.
- **replace_all:** use for rename-across-file operations (variable rename, import path change, etc.).
- **Noop edit:** FAIL with `No changes to make: old_string and new_string are exactly the same`. Re-read the file before retrying — the actual content likely differs from what was assumed (external edit, indentation mismatch).
- **File modified since read:** FAIL with `File has been modified since read, either by the user or by a linter`. Re-Read the file explicitly before retrying. Happens when a linter, worker, or user edits between your Read and Edit calls.

### Write
- **Existing file:** Read first required — see Rule 9.
- **Edit over Write:** for existing files, prefer Edit (sends only the diff). Write sends the full content every time.
- **No docs:** NEVER create `*.md` or README files unless explicitly requested by the User.

---

