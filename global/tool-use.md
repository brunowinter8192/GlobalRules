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


### 2. Context window hygiene — verbose output to file, not context

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

- NEVER run `cat` on a file that might be large — use `head`, `tail`, `grep`

### 3. Stop after 2 failed tool calls

When 2 tool calls in a row fail or don't deliver the desired result: **STOP IMMEDIATELY**.

- Clearly explain the problem to the user
- Ask: "How should I solve this?" or "Where can I find X?"
- NO further trial and error without user input
- After 2 failures: the answer is RESEARCH (Web/GitHub search, read source code, read docs), not RETRY


### 4. One Bash tool_use block per response — chain inside it

**Correct dispatch pattern:** each assistant response carries at most ONE Bash tool_use block. Put every Bash-class action for that turn INSIDE that single block, chained with `;` or `&&` — no limit on command count.

- Independent commands → chain with `;`
- Dependent commands (each step prerequisite for the next) → chain with `&&`
- Diagnostic chains where step failure must NOT abort the rest → `;` (see Rule 8)

Sequential-across-turns is the right move only when a LATER command needs the OUTPUT of an EARLIER one.

Applies to ALL Bash invocations. Read/Write/Edit may be sequenced together in one response when there is a genuine ordering need (e.g. Read→Edit per Rule 7); a Bash block always travels alone.

**Chain everything chainable.** When dispatching a Bash call, identify what other Bash-class actions are the obvious next step and pack them into the same block:

- After `git merge`: chain post-merge verification (`; rag-cli search_hybrid "test" rag-cli-docs`)
- After `worker-cli send X`: chain status check of other workers (`; worker-cli list`)
- After identifying a bug via investigation: chain the fix-dispatch (`; worker-cli send X "fix Y"`)
- Cleanup actions (`rm /tmp/scratch.*`, `git stash drop`) cost zero extra tool calls when chained — always chain them.

**Verbal-deferral is forbidden.** Phrases like "I'll do X next turn" / "Timer setze ich gleich" / "ich verifiziere das anschließend" / "Mache ich später" trigger an immediate self-check: **could X have been chained into the current Bash call?**

- If YES → rule violation. Rewrite the response, chain the action.
- If NO (genuine tool-constraint: background-Bash + foreground-Bash conflict, Read-required-before-Edit that did not fit, etc.) → state the explicit constraint AND specify the exact next-turn first-action as a concrete command, not a vague promise.

Investigation, follow-up, verification all count as chainable same-turn actions: `worker-cli response`/`status`, `grep` on an identified pattern, `rag-cli` verify, post-merge tests, diff review, issue comment, file delete. These are NEVER next-turn material when current Bash budget is available.


### 5. Tool failure → immediate action

Tool call fails → do NOT continue with workaround or fallback without reporting.

1. Tool fails → report error to user in same message
2. Can fix it (start process, install dep, create dir)? → fix NOW, retry
3. Can't fix it (needs user credentials, hardware, manual step)? → stop, explain what's needed
4. NEVER silently switch to plan B without disclosing plan A failed
5. NEVER ask "should I start X or work around it?" — if you CAN fix it, fix it


### 6. `<persisted-output>` blocks: ALWAYS Read the full file — never grep, never preview

When a tool_result contains a `<persisted-output>` block (`Full output saved to: <path>`), ALWAYS Read the full file via the Read tool — one Read call on the path. NEVER grep it, NEVER head/tail/cat-filter it, NEVER offset/limit-chunk it, NEVER settle for the `Preview (first NKB)` content.

**Workflow:**
1. Extract the absolute path from `Full output saved to: <path>`.
2. Read the ENTIRE file in ONE Read call. If it exceeds 2000 lines, raise `limit=N` to cover it.

**Only exception — physical Read-tool limits:** the Read tool hard-fails on files >256KB or >25k tokens. ONLY in that case fall back to a targeted `grep -n <target> <path>` to find the line, then a bounded Read with `offset`/`limit` around it. Typical persisted tool/RAG outputs are tens of KB — this exception almost never fires. When in doubt, Read the whole file.


### 7. Read before Edit/Write — non-negotiable

Before any Edit or Write call on an existing file, call Read on the same path. ONE Read per file per session is enough — subsequent Edit/Write calls reuse the read state. Read followed by Edit in the same response is fine; both fire in order.

**Forbidden shortcut:** "I know the content already" or "I just edited this in another session" does NOT satisfy. Read-state is per-session, not persisted across sessions.


### 8. Diagnostic Bash chains: `;` not `&&`

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


### 9. zsh Quoting for Repeated Path Calls

When a command is called repeatedly with the same long paths, NEVER pack two paths into a single variable with whitespace. Three robust patterns:

1. **Function:** `cmd() { /full/python /full/cli.py "$@"; }`
2. **Two-Variable-Split:** `PY=/full/python; CLI=/full/cli.py; $PY $CLI ...`
3. **Wrapper script in `~/.local/bin/`:** 3-line bash wrapper + `chmod +x`, then just `cmd-name ...` (pattern: rag-cli, reddit-cli, arxiv-cli)


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

#### Git CLI

Commit primitives. Push / merge / plugin-publish / multi-repo are orchestrator-only → see `opus/tool-use.md` § Git CLI.

##### Pre-Commit

`git-check [repo_path]` — `repo_path` accepts `c` (project-path shortcut). Auto-stages files (with skip patterns: venv/, node_modules/) and returns a status report:
- `STAGED` / `UNSTAGED` / `UNTRACKED` sections
- `HOOK STATUS`
- `DIFF SUMMARY` → use for commit message

If all sections are `(none)` → nothing to commit, skip.

##### Commit Commands

| Operation | CLI | Notes |
|---|---|---|
| Commit (inside repo/worktree cwd) | `gc "<message>"` | Wrapper: stages tracked modifications + commits. Add filenames as extra args to stage specific files. |
| Commit (explicit path) | `git -C <repo_path> commit -am "<message>"` | Use when cwd is outside target repo. `-am` stages tracked mods. For untracked: `git -C <path> add <files> && git -C <path> commit -m "<msg>"`. |
| Post-commit check | `git -C <repo_path> status --short` | Empty output = clean working tree. |

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

##### Rules (Safety Protocol)

- If push fails → report error, do NOT retry
- Commit only when explicitly asked

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
- **Empty file:** returns a system-reminder warning in place of content — do NOT interpret the warning as actual file content.
- **25k-token limit:** files >25k tokens fail with `File content (X tokens) exceeds maximum allowed tokens (25000)`. Same fix: grep + targeted Read with offset/limit.
- **Nonexistent path:** fails with `File does not exist. Note: your current working directory is …`. Verify path with `ls` before Read when path is reconstructed from memory. Common typos: `.claire/` (should be `.claude/`), `..claude/` (double-dot — never valid).
### Edit
- **Read first:** Required — see Rule 7.
- **Indentation:** preserve EXACT indentation as it appears in the file content.
- **Uniqueness:** FAIL if `old_string` is not unique in the file. Remedy: expand the match string with more surrounding context, or use `replace_all`.
- **replace_all:** use for rename-across-file operations (variable rename, import path change, etc.).
- **File modified since read:** FAIL with `File has been modified since read, either by the user or by a linter`. Re-Read the file explicitly before retrying. Happens when a linter, worker, or user edits between your Read and Edit calls.

### Write
- **Existing file:** Read first required — see Rule 7.
- **Edit over Write:** for existing files, prefer Edit (sends only the diff). Write sends the full content every time.
- **No docs:** NEVER create `*.md` or README files unless explicitly requested by the User.

---

