# Tool-Use

## Core Rules

### Stop after 2 failed tool calls

When 2 tool calls in a row fail or don't deliver the desired result: **STOP IMMEDIATELY**.

- Clearly explain the problem to the user
- Ask: "How should I solve this?" or "Where can I find X?"
- NO further trial and error without user input

## Bash

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

Noisy outputs go through files; signal outputs go directly into context. Noisy → redirect to a `/tmp` file, then grep/tail the signal.

```bash
./venv/bin/python dev/crawling_suite/03_test.py > /tmp/03_test_output.md 2>&1
tail -20 /tmp/03_test_output.md
```

### Chaining Bash calls

Chain everything you can into ONE Bash tool_use block per response. The one legitimate reason to split across turns: the follow-up call requires you to INTERPRET an earlier call's output and decide from it what to run next. Data-dependency alone is NOT a reason. Don't verbally defer ("I'll do X next turn") what could have chained into the current block.

Read/Write/Edit may be sequenced together in one response when there is a genuine ordering need (e.g. Read→Edit); only the Bash block travels alone.

### zsh Quoting for Repeated Path Calls

When a command is called repeatedly with the same long paths, NEVER pack two paths into a single variable with whitespace. Use a wrapper script in `~/.local/bin/`: a 3-line bash wrapper + `chmod +x`, then just `cmd-name ...` (pattern: rag-cli, reddit-cli, arxiv-cli).

### Git CLI

Commit with `gcommit "<message>" [repo_path]`: stages all changes (tracked mods + untracked, minus a secrets skip-list) and commits in ONE call, worktree-correct — commits on the current worktree's branch, never the parent repo. `repo_path` defaults to cwd. Push is separate and orchestrator-only.

#### Commit Message

Single-line, type-prefixed (`feat` / `fix` / `refactor` / `docs` / `chore`), one concern per commit.

- Max 72 chars
- Pick dominant concern if mixed
- No Co-Author footer for routine commits

## Read

Reads up to 2000 lines by default; `cat -n` format (`line_number\tcontent`). Use `offset`+`limit` for larger files, `pages` for PDFs (REQUIRED on PDFs >10 pages, max 20/request — omitting fails the call). Reads images (PNG/JPG) and `.ipynb` notebooks.

- **Nonexistent path:** fails with `File does not exist…`. Verify with `ls` when reconstructing a path from memory. Common typos: `.claire/` (should be `.claude/`), `..claude/` (double-dot — never valid).

### Grep for patterns, Read for meaning

Grep ONLY when the target is a fixed, unambiguous pattern — a symbol, import, path, literal string, exact token (typically code). When the target is semantic — whether a topic is covered, a claim is made, an idea is present — READ the file (offset/limit for large ones), never grep. Prose says the same thing many ways: grepping `haus` returns nothing when the file says `villa` — a false negative. Meaning, synonyms, anything without a fixed pattern → read, not grep.

### `<persisted-output>` blocks: ALWAYS Read the full file — never grep, never preview

When a tool_result contains a `<persisted-output>` block (`Full output saved to: <path>`), ALWAYS Read the full file via the Read tool — one Read call on the path. NEVER grep it, NEVER head/tail/cat-filter it, NEVER offset/limit-chunk it, NEVER settle for the `Preview (first NKB)` content.

**Workflow:**
1. Extract the absolute path from `Full output saved to: <path>`.
2. Read the ENTIRE file in ONE Read call. If it exceeds 2000 lines, raise `limit=N` to cover it.

## Edit

Performs exact string replacement in a file.
- You must Read the file in this conversation before editing, or the call will fail. ONE Read per file per session suffices — later Edits reuse the read-state. Read-state is per-session: "I know the content already" / "I edited it last session" does NOT satisfy; Read again.
- `old_string` must match the file exactly, including indentation, and be unique — the edit fails otherwise. Strip the Read line prefix (line number + tab) before matching.
- `replace_all: true` replaces every occurrence instead.
- **File modified since read** (linter / worker / user touched it between your Read and Edit) → re-Read before retrying.

## Write

Writes a file to the local filesystem, overwriting if one exists.
- When to use: creating a new file, or fully replacing one you've already Read. Overwriting an existing file you haven't Read will fail. For partial changes, use Edit instead.
- **Prefer Edit** for existing files — Write resends the full content every time.
