# Tool-Use

## Core Rules

### Stop after 2 failed tool calls

**Stop immediately when two tool calls in a row fail or miss the goal.**
Blind retries burn context and rarely converge, so a third attempt is banned until the user weighs in.

- clearly explain the problem to the user
- ask "how should I solve this?" or "where can I find X?"
- no further trial and error without user input

## Bash

**Bash never writes repo files — use Write/Edit for any file that lives in the repo (`.py`, `.sh`, `.md`, configs).**
The only bash-written files are throwaways under `/tmp/`; a shell-argument heredoc for a one-shot command (commit body, curl payload) is fine, since it produces no persistent file.

### Context window hygiene — verbose output to file, not context

**Route verbose output to a file, keep only the signal in context.**
Noisy command output goes to a `/tmp` file; then grep or tail just the signal back into context.

```bash
./venv/bin/python dev/crawling_suite/03_test.py > /tmp/03_test_output.md 2>&1
tail -20 /tmp/03_test_output.md
```

### Chaining Bash calls

**Chain everything you can into ONE Bash block per response.**
The only reason to split across turns is when the follow-up call needs you to INTERPRET an earlier call's output to decide what to run next; data-dependency alone is not a reason.

**Never verbally defer what could have chained into the current block.**
Don't say "I'll do X next turn" for a call that no dependency forced into a later turn.

**Read/Write/Edit may be sequenced in one response when ordering demands it (e.g. Read→Edit).**
Only the Bash block travels alone.

### Git CLI

**Commit with `gcommit "<message>" [repo_path]` — it stages all changes and commits in ONE call, on the current branch.**
It stages tracked mods plus untracked (minus a secrets skip-list) and commits on the current branch of wherever you are — a worktree's branch when in one, the repo's own branch when working directly in it, never the parent repo. `repo_path` defaults to cwd. Push is separate and orchestrator-only.

#### Commit Message

**Single-line, type-prefixed, one concern per commit.**
Prefix with `feat` / `fix` / `refactor` / `docs` / `chore`.

- max 72 chars
- pick the dominant concern if mixed
- no Co-Author footer for routine commits

## Read

**Reads a file in `cat -n` format (`line_number\tcontent`).**
Use `offset`+`limit` for larger files, `pages` for PDFs (REQUIRED on PDFs >10 pages, max 20/request — omitting fails the call). Reads images (PNG/JPG) and `.ipynb` notebooks.

**A nonexistent path fails with `File does not exist…`.**
Verify with `ls` when reconstructing a path from memory — common typos: `.claire/` (should be `.claude/`), `..claude/` (double-dot, never valid).

### Grep for patterns, Read for meaning

**Grep only for a fixed, unambiguous pattern; Read when the target is meaning.**
Grep fits a symbol, import, path, literal string, exact token — typically code. When the target is semantic — whether a topic is covered, a claim is made, an idea is present — READ the file (offset/limit for large ones), never grep, because prose says the same thing many ways: grepping `haus` returns nothing when the file says `villa`, a false negative.

### `<persisted-output>` blocks: ALWAYS Read the full file — never grep, never preview

**A `<persisted-output>` block (`Full output saved to: <path>`) is always Read in full, never grepped or previewed.**
One Read call on the path — never grep it, never head/tail/cat-filter it, never offset/limit-chunk it, never settle for the `Preview (first NKB)` content.

**Workflow:**
1. extract the absolute path from `Full output saved to: <path>`
2. Read the ENTIRE file in ONE Read call; if it exceeds 2000 lines, raise `limit=N` to cover it

## Edit

**Performs exact string replacement in a file.**
Read the file in this conversation before editing, or the call fails.

- one Read per file per session suffices — later Edits reuse the read-state
- `old_string` must match the file exactly, including indentation, and be unique — strip the Read line prefix (number + tab) before matching
- `replace_all: true` replaces every occurrence instead

## Write

**Writes a file to the local filesystem, overwriting if one exists.**
Create a new file or fully replace one you've already Read — overwriting an unread file fails, and for partial changes you use Edit.

- **prefer Edit** for existing files — Write resends the full content every time
