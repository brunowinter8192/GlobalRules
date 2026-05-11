# Tool-Use

Goal: no large Bash-call where a small one works. Every tool_use input counts — tool_use JSON inflates with escaped newlines and quotes. Live feedback in Monitor_CC waste_pane (Window 4 right) shows current-session ratios ≥3.

## Hard Rules — Token Efficiency

### 1. Python: heredoc for one-shot, Write + Exec ONLY for iteration (binary rule)

**This is not a style preference. It is a measurable cost difference per probe:**

- One-shot probe via **heredoc** = **1 tool call** (`python3 << 'EOF' ... EOF`)
- Same one-shot via **Write + Exec** = **2 tool calls** (Write the file, then Bash to run it)

Write + Exec on a one-shot probe is pure waste — one extra tool call, one extra `tool_use` JSON payload, one extra `tool_result`, and a temp file left behind. The iteration-discount of Write + Edit **only** pays from run #2 onward, when Edit diffs replace full-heredoc re-transmission. There is no third option and no "Faulheit" / "cleaner / easier" justification — the call-count delta is the rule.

**Decision is binary, by reuse:**

1. **jq / grep / awk first** — purpose-built, shortest form. `jq -c 'select(.type=="error")' file.jsonl` beats 15 lines of Python every time. Use a one-liner when the data shape fits.
2. **Python heredoc** when Python is actually needed (multi-field comparisons, nested dict walks, shapes awk/jq struggle to express). One run, one heredoc, done.
3. **Write + Edit** ONLY when the SAME script will run a second time after edits. Switch from run #2, not before.

**Different script for a different question is a NEW one-shot.** Five probes answering five questions in one session = five heredocs, not five Writes. The signal is "will this exact script run again with changes" — not line count, not "it looks substantial", not "a file feels cleaner".

**`python3 -c`:** when the `-c` string exceeds 300 chars including escaping — switch to heredoc (or Write once iteration starts). Argument-level quote escaping in `-c` is worse than heredoc quoting for medium scripts.


### 2. No Bash for file creation → Write tool

**Rule:** NEVER use `cat > file << 'EOF'` or `echo >` to create files. Always use the Write tool.

**Why:** Bash heredocs can leak shell context into file content (e.g., `EOF 2>&1 | head -10` appended to a .gitignore). The Write tool is atomic and safe.

**Exception:** Single-line echo append to an existing file is fine: `echo "entry" >> config.log`.

### 3. Grep scope hygiene — always restrict when searching source

`grep -rn <pattern> <dir>` without type/include restriction matches inside JSONL, log files, vendored content, and node_modules. Output can explode into 10+ MB of irrelevant matches, poisoning context.

**Rule:**
- When searching for Python imports, function refs, or code-level patterns: always pass `--include='*.py'` to bash grep OR use the Grep tool with `type: "py"` / `glob: "*.py"`.
- Prefer the Grep tool over bash `grep -rn` for code search — safer defaults, structured results.
- For one-off bash grep: add explicit file scope (`grep -n <pattern> <specific_file>`) rather than `-r` over a whole tree.


### 4. Context window hygiene — verbose output to file, not context (CRITICAL)

Large tool outputs (build output, test runners, dev scripts, background tasks) flood the context window. One verbose dump can burn 10k+ tokens of irrelevant noise.

**Rule: noisy outputs go through files; signal outputs go directly into context.**

The split:

**File-redirect path (NOISY OUTPUT — most output is irrelevant):**
- Build output (compiler errors mostly drowning in green-passes)
- Test runners (mostly green dots, you grep for FAIL)
- Dev scripts with debug logs you sample
- Background workflows you tail-check
- Anything where you want one specific signal out of a verbose dump

```bash
# RIGHT for noisy output — redirect, then extract
./venv/bin/python dev/crawling_suite/03_test.py > /tmp/03_test_output.md 2>&1
tail -20 /tmp/03_test_output.md
```

**Direct-to-context path (SIGNAL OUTPUT — the output IS the answer):**
- Search-CLI results (searxng-cli search_web / search_batch — the URLs/snippets are the data you need to evaluate as a whole)
- RAG retrieval / MCP tool results returning structured data
- Single-file reads via `cat` / `head` / `tail` of bounded size
- Git status, log, diff (when bounded)
- Anything where you'd just re-read the file in chunks anyway, ending up with the same content in context

```bash
# RIGHT for signal output — direct call, full result lands in context
searxng-cli search_batch "query 1" "query 2" "query 3" "query 4"
# Up to 4 queries × 20 URLs = 80 results, ~20 KB / ~5K tokens. Fits in one tool result.
```

**The test before redirecting:** is the output mostly noise you'll grep through, or is it the data you'll evaluate as a whole? Noise → file. Data-as-whole → direct.


- NEVER run `./venv/bin/python script.py` without `> /tmp/file.md 2>&1` (dev scripts are noisy)
- NEVER run `cat` on a file that might be large — use `head`, `tail`, `grep`
- DO run search/RAG/MCP-tool calls in the foreground — the result is what you came for

### 5. Stop after 2 failed tool calls

When 2 tool calls in a row fail or don't deliver the desired result: **STOP IMMEDIATELY**.
- Clearly explain the problem to the user
- Ask: "How should I solve this?" or "Where can I find X?"
- NO further trial and error without user input

**"Quellen" = External Research, NOT More Bash:**
- After 2 failures: the answer is RESEARCH (Web/GitHub search, read source code, read docs), not RETRY
- Same error in a different wrapper = same bug. After 2nd failure: analyze the error pattern, ask what the common denominator is.


### 6. Never dispatch parallel Bash calls

Multiple Bash tool_use blocks in the same turn get serialized by the runtime — one wins, the others come back as `<tool_use_error>Cancelled: parallel tool call Bash(...)</tool_use_error>`. The cancelled calls still cost input tokens, produce zero useful output, and force a retry. Pure waste.

**Rule:** one Bash call per turn. If you need multiple bash operations:
- Chain them in a single command with `&&` or `;` (when outputs can be combined)
- Or run sequentially across turns (when each result informs the next)

Applies to ALL Bash invocations, not just `ls` — git, grep, cat, worker-cli, anything. Other tools (Read, Write, Edit, Grep, Glob) can be dispatched in parallel safely; only Bash has this cancel behavior.


### 7. Tool failure → immediate action (CRITICAL)

Tool call fails silently → do NOT continue with workaround or fallback without reporting.

**Rule:**
- Tool fails → report to user IMMEDIATELY in the same response
- Then: (a) fix the prerequisite yourself (start server, install dep, fix config) and retry, OR (b) stop and wait for user input if fix is outside your control
- NEVER silently fall back to a different approach without disclosing plan A failed
- NEVER ask "should I start X or work around it?" — if you CAN fix it, fix it

**Decision tree:**
1. Tool fails → report error to user in same message
2. Can I fix it? (start process, install dep, create dir) → fix it NOW, retry
3. Can't fix it? (needs user credentials, hardware, manual step) → stop, explain what's needed
4. NEVER: silently switch to plan B without disclosing plan A failed


### 8. `<persisted-output>` blocks: grep the full file, never settle for the preview

**Rule:** When a tool_result contains a `<persisted-output>` block (CC's truncation feature for large Bash outputs), use Grep / Read / cat on the persisted file path. NEVER stop at the `Preview (first NKB)` content.

CC injects this format when a Bash output exceeds its inline limit:

```
<persisted-output>
Output too large (NMB). Full output saved to: /Users/.../tool-results/<id>.txt

Preview (first 2KB):
... 2KB snippet ...
...
</persisted-output>
```

The preview is bait. It suggests "this is everything you can see" and the natural read is to draw conclusions from those 2KB alone. That is almost always wrong — the full data lives at the path and is one Grep / Read call away. Preview content is redundant the moment you grep the persisted file.

**Workflow:**
1. Extract the absolute path from `Full output saved to: <path>`.
2. **Grep first** — `grep <pattern> <path>` for targeted lookups (lowest context cost).
3. **Read with offset/limit** for ranges — `Read(file_path=<path>, offset=N, limit=M)`. CC's "too large" warning is conservative; direct Read on the persisted file works for files much larger than the inline tool-output limit.
4. **cat / head / tail** only when the file is small and you genuinely need contiguous content.

**The mistake to avoid:** answering questions, drawing conclusions, or planning next steps from preview content alone. If preview content looks like it answers the question, that is coincidence — the full file may contain the actual answer or a different signal entirely.


**>100KB persisted: don't re-grep on the persisted file.** When an initial Bash call produces a persisted-output >100KB (typical with recursive grep over `~/.claude/` matching plugin cache + JSONL session files), a follow-up grep on the same persisted file with the same pattern produces almost the same size — persisted again, too large for Read. Don't drill down on the over-broad pattern. Instead: (a) tighten the pattern (what was too broad?), (b) narrow scope (which files explicitly?), (c) re-run with narrow scope from the start.

**Don't chunk-read small persisted files.** When the persisted file is up to roughly 100 KB / ≤2000 lines, ONE Read call covers the whole content (Read's default limit is 2000 lines, raise via `limit=N` if needed). Reading in 100/200/rest chunks across three sequential Read tool_use blocks is pure overhead — three roundtrips, three tool_result payloads, for content that fits in one. Chunked reading is only justified for genuinely huge files (hundreds of MB, log archives) where a full Read would itself blow the context window. Decision rule before Read: estimate file size from the persisted-output header (`Output too large (NMB)`); if the size is under ~200 KB, read it all at once with `limit=` set generously, not in increments.


### 9. Read before Edit/Write — non-negotiable

The Edit and Write tools fail on files that haven't been read in the current session with `<tool_use_error>File has not been read yet. Read it first before writing to it.` There is no workaround — the call must be re-issued after a Read.

**Rule:** before any Edit or Write call on an existing file, call Read on the same path. ONE Read per file per session is enough — subsequent Edit/Write calls reuse the read state. Read followed by Edit in the same response is fine; both fire in order.

**Forbidden shortcut:** "I know the content already" or "I just edited this in another session" does NOT satisfy the requirement. The Read-state is per-session, not persisted across sessions.

The per-tool reference at the bottom of this file (Edit / Write sections) carries the same rule, but it gets overlooked because it lives in a reference table rather than the Hard Rules. This is a Hard Rule.


### 10. Branch-name ambiguity in repos with same-named directories

`git diff <branch>` and `git log <branch>` fail with `fatal: ambiguous argument '<branch>': both revision and filename` when a directory of the same name exists at the repo root. The classic case: a `dev/` directory (for dev-scripts, evals, etc.) plus a `dev` branch. Git cannot tell which you meant.

**Rule:** when the branch name collides with a directory in the repo, disambiguate with a trailing `--` to force the branch interpretation:

```bash
git -C <repo> diff dev --
git -C <repo> log dev --oneline --
```

Alternatives that also work:
- `git diff origin/dev` — remote-prefixed refs never collide with directory names
- `git diff main` — compare against trunk instead, when that's what you actually want

`workers-2` prescribes `git -C <project_root>/.claude/worktrees/<name> diff dev` for code review of worker branches. In a Monitor_CC-shaped repo (with `dev/` at root) this triggers the ambiguity error every time. Use `git -C <worktree> diff dev --` or `git -C <worktree> diff main` instead.


### 11. Diagnostic Bash chains: `;` not `&&`

**Hard test before writing `cmd_a && cmd_b`:** ask whether `cmd_b` would exit non-zero in a normal correct case — e.g. `grep` no-match, `ls` on empty/missing directory, `test -f` on a path that may not exist, `wc` on missing file. If yes → use `;`.

`&&` is reserved for sequences where step N's success is a *prerequisite* for step N+1: build before test, mkdir before write, commit before push, install before run. Diagnostic chains, verification probes, multi-source sanity-checks always use `;`.

```bash
# WRONG — grep no-match aborts the rest
echo "=== refs ===" && grep X file && ls dir/ && echo "=== done ==="

# RIGHT — each step runs regardless
echo "=== refs ==="; grep X file; ls dir/; echo "=== done ==="
```

Same principle: `2>/dev/null` swallows stderr but does NOT change exit codes — adding it does not save the chain.


### 12. `sleep` commands are forbidden — single narrow exception for Opus worker-polling (NON-NEGOTIABLE)

**Hard ban: any Bash tool call containing `sleep` is FORBIDDEN, with exactly one allowed form documented below.** Observed violation pattern: workers chain-spam `sleep 180 && echo done` → `sleep 60` → `sleep 30` → `sleep 60` → `sleep 180` waiting on their own internal long-running tasks. Each iteration burns 1-2% context for zero progress, pollutes the waste-pane, and indicates the underlying task is mis-architected.

**Workers: zero sleep, ever.** No polling own background tasks, no "let the system settle", no "wait for output to appear", no "give the GPU service a moment". If a worker's task takes longer than the 10-minute `Bash` tool timeout ceiling, restructure:

- Split into shorter Bash calls (e.g. 4 queries × separate Bash call instead of one 13-min batch)
- Use foreground Bash with explicit `timeout=600000` (10-min max, the tool's hard ceiling)
- Write incremental progress to `/tmp/<name>.log`, finalize after natural completion
- Background-spawn + sleep-poll is the explicit anti-pattern — do NOT use it

The instinct "I just need to wait for X seconds and then check" is wrong. The right framing is: the work either completes within the foreground Bash call, or it gets restructured. There is no waiting.

**Opus: one allowed form, narrow.** The single exception:

```
Bash(command="sleep N && echo done", run_in_background=true)
```

- Exact form only, no variations, no additional chaining
- Used by Opus to schedule the next status-check on a dispatched **worker** (not on Opus's own background scripts, not on Opus's own pipeline runs)
- Single timer at a time (Background Task Discipline, Rule 6 / Rule 14)
- The user receives "Background command completed" as a turn input, then Opus checks `worker-cli status` foreground in the next turn

This is the canonical worker-polling flow (`opus-workers-2`). It exists because Opus needs to free the API stream while a worker runs in a separate tmux session. Workers themselves don't have this constraint — they can let foreground Bash run up to 10 min directly, no sleep needed.

**Self-check before any Bash call** (mandatory): does the command contain the literal token `sleep`? If yes:

- Worker context → DELETE the call, restructure the task per the bullet list above
- Opus context → confirm ALL THREE: exact form `sleep N && echo done`, AND `run_in_background=true`, AND polling a worker (not Opus's own pipeline). Otherwise DELETE.

**Runtime backstop:** CC's tool-use runtime already blocks `sleep N && <other_command>` (chained with anything other than `echo done`), returning `<tool_use_error>Blocked: sleep N followed by: <command>`. This is a hard backstop against the most common violation. The discipline rule above is stricter — it bans the pattern at the design level, before the runtime has to catch it.


### 13. Worktree path is `.claude/worktrees/` — never `.claire/`

The worktree directory in every project is `.claude/worktrees/<name>/`. There is no `.claire/` anywhere. But there is a recurring tokenizer-level typo where Edit, Write, Read, or Bash calls inside worker sessions land on `.claire/worktrees/...` paths and fail with `File does not exist` (or, for Bash `cd`, with a no-such-directory error).

**Rule:** before any file operation that names a worktree path explicitly, verify the literal substring `.claude/worktrees/` (with a `u` and `d`, not an `i` and `r`). When working from cwd inside a worktree, prefer relative paths or the `c` shortcut for `worker-cli` rather than reconstructing the absolute path — the typo only happens when the model rebuilds the path string by hand.

**Detection:** if a tool call returns `<tool_use_error>File does not exist. Note: your current working directory is /Users/.../.claude/worktrees/<name>.` — the cwd is correct but the path argument has the typo. The fix is rewriting the file_path with `.claude/`.

**Same-class typo: `..claude/...`** (two dots, no slash). Real paths have `..` only as `../` (relative parent traversal). Any path matching `\.\.[a-z]` (two consecutive dots immediately followed by a lowercase letter) is a typo with overwhelming probability — it is virtually never a valid path component.


### 14. Background Bash is a deliberate choice, never a default

Setting `run_in_background=true` on a Bash tool call MUST be a conscious decision — never typed by default for grep / cat / ls / read-only commands. Background mode triggers the persisted-output mechanism (Output too large notification + file path), spawns a `<task-notification>` injection on the next user turn, and creates orchestration complexity (the next turn's REQ may carry a TN block + a system-notification SR depending on CC version).

**Use background ONLY for:**
- Long-running commands the user explicitly wants to overlap with other work (build/test runs >30s, `sleep N && echo done` as orchestration timer)
- Worker-spawn timer patterns documented in opus-workers-2

**Never use background for:**
- Quick grep / cat / ls / wc / git status / file inspection
- Anything you'll read the output of within the same response
- "Just to be safe" / "in case it takes long"


### 15. zsh-Quoting bei wiederholten Pfad-Aufrufen

bash splittet `$VAR` mit Whitespace beim Erweitern in Argumente. zsh **nicht** — `$A` bleibt ein einzelnes Argument auch wenn der String Leerzeichen enthält. Resultat: `A="/path/python /path/cli.py"; $A search ...` versucht in zsh ein Programm namens `/path/python /path/cli.py` (mit Leerzeichen im Namen) zu starten und scheitert mit `no such file or directory`. Auf macOS ist zsh die Default-Shell.

**Rule:** wenn ein Befehl mehrfach mit denselben langen Pfaden aufgerufen wird, NIE zwei Pfade in eine einzelne Variable mit Whitespace packen. Drei robuste Patterns:

1. **Function:** `cmd() { /full/python /full/cli.py "$@"; }` — `"$@"` ist Pass-Through, splittet sicher.
2. **Two-Variable-Split:** `PY=/full/python; CLI=/full/cli.py; $PY $CLI ...` — jede Variable enthält genau einen Pfad ohne Whitespace.
3. **Wrapper-Script in `~/.local/bin/`:** Ein 3-Zeilen-Bash-Wrapper (`#!/usr/bin/env bash` + `exec /full/python /full/cli.py "$@"`), `chmod +x`, dann nur noch `cmd-name search ...`. Pattern bei rag-cli, gh-cli, reddit-cli, arxiv-cli.


### 16. cd-Drift across Bash-Tool-Calls

Bash-Tool-Calls in einer Session teilen die cwd. Ein `cd /target` in Call N ändert die cwd für Call N+1, N+2 usw. Falle, wenn Diff-Reviews oder Worktree-Operationen mitten in einer ansonsten main-repo-zentrierten Session laufen.

**Rule:** wenn ein Bash-Call `cd "$WORKTREE"` oder ähnlichen Direktwechsel enthält, MUSS der letzte Schritt im selben Call zurück in die main cwd cd'en (`cd /full/main/repo/path` am Ende). Alternativ: durchgängig `git -C <path>` und absolute Pfade, ohne überhaupt zu cd'en.


---

## Soft Rules

### Repeated absolute paths → env var or single `cd`

When a path appears in 3+ consecutive commands: use `$MONITOR_CC_ROOT` (or equivalent) or `cd` once at the start of a planned block. Do NOT drift `cd` silently across interactive steps — only within a contained sequence.

### Don't chain greps/cats over the same pattern

```
WRONG: grep X a.log && grep X b.log && grep X c.log
RIGHT: grep X a.log b.log c.log
```

### Don't re-issue near-identical commands

If you've run a command in this session and the output wasn't what you needed, do NOT retry with a minor variation. Change approach: different tool (Grep/Glob), different scope, or read source.

### Grep/Glob gunshot

Multiple Grep/Glob calls with varied patterns that all return zero results = guessing. Pattern:
1. `Glob` first: find files matching a broad path pattern.
2. `Grep` on the hit: targeted pattern on a known file.

Zero-results live in the warnings_pane (Monitor Window 4 left). Two zero-results in a row on the same topic = stop, rethink.

---

## Large Artifacts

### Heredoc — three distinct cases

Not every heredoc is the same problem. Three classes, three different answers.

**Case 1 — Python / analysis: one-shot = heredoc. Iteration = Write + Edit. Binary.**

`python3 << 'EOF' ... EOF` is the form for a one-shot probe — 1 tool call, no temp file.

Switch to Write + Edit the moment you're about to run the SAME script a second time after editing it. Not third run, not fourth — from run #2. The switch pays because Edit diffs are smaller than re-transmitting the full script.

Different script for a different question is NOT iteration. It is a new one-shot → new heredoc. Running five distinct probes in one session means five heredocs, not five Writes.

Still: jq / awk / sed first when the question fits a one-liner. Python (via heredoc) is for shapes those don't express cleanly.

**Case 2 — File creation heredoc (Rule 2 above): NEVER.**

`cat > file << 'EOF'` and `echo >` can leak shell context into the file content (e.g. `EOF 2>&1 | head -10` accidentally appended). Write tool is atomic and safe. Zero exceptions except the narrow single-line-append pattern noted in Rule 2.

**Case 3 — Shell-argument heredoc for a one-shot command: OK.**

Bead description, multi-line git commit body, one-off `curl -d`-style payload. The content is a single argument to a single command, never executed again, never edited. The alternatives (Write tool + `Bash` with `$(cat /tmp/file)`) and heredoc inline carry the **same content bytes** in tool_use JSON — the only difference is the heredoc version skips one tool-call overhead and does not leave a temp file behind.

```bash
# OK — one-shot shell argument
bd --repo <path> create --title "..." --type task --description "$(cat <<'EOF'
<full markdown description>
EOF
)"
```

Case 3 applies specifically to multi-line shell-command arguments that the user will never iterate on. For anything that might be re-run, re-shaped, or debugged later, fall back to Write + `$(cat ...)` so the content is editable via the Edit tool.

**Decision flow:**

1. Is the content Python / analysis code? → try jq/awk/sed first. If Python is needed: one-shot (one run, throw away) → heredoc. Same script will run a second time after edits → Write + Edit from run #2.
2. Is the goal to create a file? → never heredoc. Write tool.
3. Is it a multi-line argument to a one-shot shell command (bd create, git commit -m body, curl -d payload)? → heredoc inline is the clean form. One tool call, no temp file.
4. Will the same multi-line content be reused, revised, or referenced across multiple calls? → treat as iterated. Write + Edit.

### Bead descriptions

Case 3. Bead descriptions are written once and not iterated.

```bash
bd --repo <path> create --title "..." --type task --description "$(cat <<'EOF'
<full markdown description>
EOF
)"
```

If `bd` later grows a `--description-file` flag, Write + flag is equivalent.

### Git commit messages

Single-line `-m` is the default for routine commits (see `Commit Message Format` below).

Multi-line body only when it genuinely adds information for `git log` readers — and when it does, Case 3 applies: heredoc inline, no temp file.

```bash
git -C <repo> commit -am "$(cat <<'EOF'
refactor: migrate X from Y to Z

Breaking: consumers of Y must update to new signature (see MIGRATION.md).
EOF
)"
```

Multi-line body is justified when all three hold: breaking or architecturally significant change, body adds real information beyond the subject, `git log` readers benefit from the extra context. Otherwise single-line `-m` — heredoc for routine fixes is waste.

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
| Spawn worker in worktree | `worker-cli spawn <name> <prompt_file> <project_path> [model]` |

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
- `HOOK STATUS` (WARNING → run `bd export` via Bash before committing)
- `DIFF SUMMARY` → use for commit message

If all sections are `(none)` → nothing to commit, skip.

##### CLI Commands

| Operation | CLI | Notes |
|---|---|---|
| Commit (inside repo/worktree cwd) | `gc "<message>"` | Wrapper: stages tracked modifications + commits. Add filenames as extra args to stage specific files. |
| Commit (explicit path) | `git -C <repo_path> commit -am "<message>"` | Use when cwd is outside target repo. `-am` stages tracked mods. For untracked: `git -C <path> add <files> && git -C <path> commit -m "<msg>"`. |
| Push (NON-plugin repo) | `git -C <repo_path> push` | Falls back to `-u origin <branch>` if no upstream. **Use `plugin-publish` if `.claude-plugin/plugin.json` exists.** |
| Push with upstream (NON-plugin repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | For first push on new branch. |
| Post-commit check | `git -C <repo_path> status --short` | Empty output = clean working tree. `.beads/` entries can be treated as clean. |
| Push (PLUGIN repo) — replaces `git push` | `cd <plugin-source-repo> && plugin-publish` | One-step: git push + cache-sync + version-bump + MCP-server-restart. **Always use this for any repo with `.claude-plugin/plugin.json`.** Never plain `git push` on a plugin repo — cache stays stale. See `situational/plugins.md`. |

##### Commit Flow

When user asks to commit:

1. **Check + Stage** — `git-check [repo_path]`
2. **Commit** — `gc "<message>"` (if cwd inside repo) OR `git -C <repo> commit -am "<message>"` (explicit path)
3. **Post-check** — `git -C <repo> status --short` → empty = proceed; non-empty with non-`.beads/` paths → stage + commit again
4. **Push** — first check: does `<repo>/.claude-plugin/plugin.json` exist?
   - **YES (plugin repo):** `cd <repo> && plugin-publish` — does git push + cache-sync + version-bump + MCP-restart. NEVER `git push` here, the cache would stay stale.
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
| Search dense | `rag-cli search <query> <collection> [--top-k N] [--document PATTERN]` |
| Search hybrid | `rag-cli search_hybrid <query> <collection> [--top-k N] [--no-rerank]` |
| Search BM25 | `rag-cli search_keyword <query> <collection> [--top-k N] [--document PATTERN]` |
| Read context | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Delete | `rag-cli delete --collection <name> [--document <doc>]` |
| Server preset | `rag-cli server {status\|list\|start\|stop\|restart} [name]` |
| Server arbitrary | `rag-cli server start --model PATH --port N --mode {embedding\|rerank} [--name LABEL]` |
| Server by port | `rag-cli server {stop\|restart} --port N` |

##### Rules

- NEVER start `llama-server` oder splade direkt. `rag-cli server start <preset>` oder arbitrary-start verwenden.
- NEVER GPU-Prozesse außerhalb `rag-cli` killen. `rag-cli server stop <preset>` oder `stop --port N` verwenden.
- Direkt search-Befehl absetzen, kein vorheriges `rag-cli server start` nötig.
- Bei persisted-output: File komplett in EINEM Read-Call lesen, kein offset/limit Chunking.
- Indexierte Collections in `data/documents/<collection>/` → rag-cli. Lokale Source-Files → Read-Tool, nicht rag-cli.

##### RAG: Multi-Model Awareness

The RAG box exposes multiple model variants per class (embedding, reranker) plus splade. The full preset list is dynamic — never assume the legacy names (`embedding`, `reranker`, `splade`) cover everything; they are prefixes.

Discovery:

```bash
rag-cli server presets         # human-readable list of all configured presets
rag-cli server presets --json  # JSON for scripts
rag-cli server status          # which preset(s) running + health
rag-cli server list            # all running servers + idle countdown
```

`rag-cli server presets` shows: name, mode (embedding/rerank/splade), model_path, default_port, and a `default` flag (true = used by `rag-cli server start` without a name + by `ensure_ready` for search/index workflows).

Switching a variant:

```bash
rag-cli server stop embedding-8b
rag-cli server start embedding-0.6b
```

Client-side `find_server_url("embedding")` does a prefix-match across all running servers — `embedding-0.6b` will then serve search requests until you switch back. Same for `reranker`. Splade has only one variant.

Anti-patterns:
- Assuming `embedding` / `reranker` / `splade` are the only valid preset names — they're prefixes. Use `rag-cli server presets` to see the full list.
- Hardcoding preset names in downstream scripts. Always pull from `rag-cli server presets --json`.
- Calling `start_arbitrary` to launch a known model variant — that bypasses preset config. Use `rag-cli server start <name>` instead.

### Grep
- **Brace escaping:** literal braces must be escaped — use `interface\{\}` to find `interface{}` in Go code. Without escaping, the pattern silently matches nothing.
- **Multiline:** by default patterns match within single lines only. For cross-line patterns (e.g. `struct \{[\s\S]*?field`), pass `multiline: true`.

### Glob
- **Sort order:** returns paths sorted by **modification time** (newest first), NOT alphabetical. First result = most recently modified file.

### Read
- **Line limit:** reads up to 2000 lines by default. Use `offset` + `limit` parameters for larger files.
- **Output format:** `cat -n` format — `line_number\tcontent`. NEVER include the `line_number\t` prefix in Edit's `old_string` or `new_string`.
- **Images:** can read PNG, JPG, etc. — presented visually as a multimodal model.
- **PDF:** files with >10 pages MUST include a `pages` parameter (e.g. `"1-5"`). Omitting it on large PDFs causes a tool failure. Max 20 pages per request.
- **Jupyter:** can read `.ipynb` notebooks — returns all cells with their outputs.
- **Directories:** Read cannot read directories. Use `ls` via Bash.
- **Empty file:** returns a system-reminder warning in place of content — do NOT interpret the warning as actual file content.

### Edit
- **Read first:** MUST call Read at least once before Edit. Tool errors if not.
- **Indentation:** preserve EXACT indentation as it appears AFTER the line-number prefix. Prefix format is `line_number\t` — NEVER include it in `old_string` or `new_string`.
- **Uniqueness:** FAIL if `old_string` is not unique in the file. Remedy: expand the match string with more surrounding context, or use `replace_all`.
- **replace_all:** use for rename-across-file operations (variable rename, import path change, etc.).

### Write
- **Existing file:** MUST call Read first. Tool fails without it.
- **Edit over Write:** for existing files, prefer Edit (sends only the diff). Write sends the full content every time.
- **No docs:** NEVER create `*.md` or README files unless explicitly requested by the User.

---

## Monitoring Self-Audit

- **waste_pane** (Monitor Window 4 right): check 1-2× per session. Expand top offenders. If the same command-prefix keeps appearing: that's a rule violation, stop and rethink.
- **warnings_pane zero-results**: repeated zero results on the same topic = Grep/Glob gunshot violation (Rule 6).
- **Per-session reports**: `dev/tool_use_analysis/YYYYMMDD_*.md` in Monitor_CC. Generated per session via ad-hoc scripts (one script per analysis, no shared library).

## What this rule does NOT do

- Does not strip tool_result content at the proxy. That's a separate concern (result-waste, tracked under a different bead).
- Does not enforce at commit time. This is advisory behavior through rule-awareness. The monitor is the feedback loop.
- Does not maintain a library or justfile. Every analysis script is one-off.
