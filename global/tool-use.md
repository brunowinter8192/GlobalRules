# Tool-Use

## Rules

### Heredoc vs Write/Edit

**Python / analysis code:**

1. **jq / grep / awk first** if the data shape fits — one-liners beat Python.
2. **Python heredoc** when Python is needed AND the script runs once.
3. **Write + Edit** ONLY when the SAME script will run a second time after edits.

Different script for a different question is a NEW one-shot — five probes answering five questions = five heredocs, not five Writes. `python3 -c`: switch to heredoc when the `-c` string exceeds ~300 chars including escaping.

**File creation or editing:** NEVER Bash heredoc / `cat > file <<EOF` / `tee file <<EOF`. Always Write (new file) or Edit (existing file) — `.py`, `.sh`, `.md`, configs, scripts, any file living in the repo. The only files you may heredoc-write are throwaways under `/tmp/`.

**Shell-argument heredoc for a one-shot command** (git commit body, curl payload): OK.

```bash
curl -X POST https://api.example.com/resource -d "$(cat <<'EOF'
{"field": "multi-line\nvalue"}
EOF
)"
```


### Context window hygiene — verbose output to file, not context

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

### Stop after 2 failed tool calls

When 2 tool calls in a row fail or don't deliver the desired result: **STOP IMMEDIATELY**.

- Clearly explain the problem to the user
- Ask: "How should I solve this?" or "Where can I find X?"
- NO further trial and error without user input


### Chaining Bash calls

Chain everything you can into ONE Bash tool_use block per response. The one legitimate reason to split across turns: a follow-up call that depends on an earlier call's output. Everything else chains inside the single block. Don't verbally defer ("I'll do X next turn") what could have chained into the current block.

Read/Write/Edit may be sequenced together in one response when there is a genuine ordering need (e.g. Read→Edit); only the Bash block travels alone.

**`;` over `&&` inside a chain.** Before writing `cmd_a && cmd_b`, ask whether `cmd_b` would exit non-zero in a normal correct case — `grep` no-match, `ls` on empty/missing directory, `test -f` on a maybe-absent path, `wc` on missing file. If yes → use `;`. `&&` is reserved for sequences where step N's success is a *prerequisite* for step N+1: build before test, mkdir before write, commit before push, install before run. Diagnostic chains, verification probes, multi-source sanity-checks always use `;`.

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


### `<persisted-output>` blocks: ALWAYS Read the full file — never grep, never preview

When a tool_result contains a `<persisted-output>` block (`Full output saved to: <path>`), ALWAYS Read the full file via the Read tool — one Read call on the path. NEVER grep it, NEVER head/tail/cat-filter it, NEVER offset/limit-chunk it, NEVER settle for the `Preview (first NKB)` content.

**Workflow:**
1. Extract the absolute path from `Full output saved to: <path>`.
2. Read the ENTIRE file in ONE Read call. If it exceeds 2000 lines, raise `limit=N` to cover it.


### zsh Quoting for Repeated Path Calls

When a command is called repeatedly with the same long paths, NEVER pack two paths into a single variable with whitespace. Three robust patterns:

1. **Function:** `cmd() { /full/python /full/cli.py "$@"; }`
2. **Two-Variable-Split:** `PY=/full/python; CLI=/full/cli.py; $PY $CLI ...`
3. **Wrapper script in `~/.local/bin/`:** 3-line bash wrapper + `chmod +x`, then just `cmd-name ...` (pattern: rag-cli, reddit-cli, arxiv-cli)


---

## Tool-Specific Reference

### Bash

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

### Read
- Reads up to 2000 lines by default; `cat -n` format (`line_number\tcontent`). Use `offset`+`limit` for larger files, `pages` for PDFs (REQUIRED on PDFs >10 pages, max 20/request — omitting fails the call). Reads images (PNG/JPG) and `.ipynb` notebooks.
- **Nonexistent path:** fails with `File does not exist…`. Verify with `ls` when reconstructing a path from memory. Common typos: `.claire/` (should be `.claude/`), `..claude/` (double-dot — never valid).

### Edit
Performs exact string replacement in a file.
- You must Read the file in this conversation before editing, or the call will fail. ONE Read per file per session suffices — later Edits reuse the read-state. Read-state is per-session: "I know the content already" / "I edited it last session" does NOT satisfy; Read again.
- `old_string` must match the file exactly, including indentation, and be unique — the edit fails otherwise. Strip the Read line prefix (line number + tab) before matching.
- `replace_all: true` replaces every occurrence instead.
- **File modified since read** (linter / worker / user touched it between your Read and Edit) → re-Read before retrying.

### Write
Writes a file to the local filesystem, overwriting if one exists.
- When to use: creating a new file, or fully replacing one you've already Read. Overwriting an existing file you haven't Read will fail. For partial changes, use Edit instead.
- **Prefer Edit** for existing files — Write resends the full content every time.
