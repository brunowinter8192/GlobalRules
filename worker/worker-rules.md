
# Worker Rules — Worktree Isolation & Report

These rules apply to every worker session. Run the Pre-Edit Check to determine your mode.

## 1. Worktree Isolation

### Pre-Edit Check (ONCE, before your first file edit)

```bash
pwd
git branch --show-current
```

**Evaluate:**
- If `pwd` contains `.claude/worktrees/` AND branch is NOT `main` → **Worktree Mode** (follow all isolation rules below)
- If `pwd` does NOT contain `.claude/worktrees/` AND branch is `main` → **Direct Mode** (worktree=false, spawned intentionally on main — skip isolation rules, edit files directly, do NOT commit)

**Pwd check before every bash block:** `pwd` must end with `.claude/worktrees/<your_name>` in Worktree Mode. If it drifted (e.g. a script `cd`'d somewhere else), `cd` back into your worktree before the next edit. Never `cd` to the main project path.

### Worktree Mode Rules

You are running in a git worktree — an isolated copy of the repo on a dedicated branch.

### Rules

1. ALL file reads and edits MUST use paths under your worktree directory.
2. NEVER use absolute paths to the main repo (the parent of `.claude/worktrees/`).
3. NEVER checkout, switch to, or commit on `main`.
4. Commit only to YOUR branch.

### Pre-Commit Check (EVERY commit)

Before every `git commit`:

```bash
git branch --show-current
```

- Expected: your worker branch name
- If it shows `main` or anything unexpected: **DO NOT COMMIT.** Something is wrong — stop and report.

### Never Commit Dependency Directories

Worktrees contain symlinked dependency directories (`venv`, `.venv`, `node_modules`) that point to the main repo's real directories. These symlinks MUST NOT be committed — they become circular self-references when merged back.

**Rule:** NEVER `git add` or commit: `venv/`, `.venv/`, `node_modules/`, or any dependency directory. Even if `git status` shows them as untracked.

Concrete failure (2026-04-03): dev-reorg worker committed `venv` symlink from worktree. After merge, project's venv became a circular symlink pointing to itself. Had to recreate venv from scratch.

## 2. Completion Checklist (MANDATORY)

Your worker prompt includes a **Completion Checklist** section — task-specific verification items defined by the orchestrator.

### How It Works

1. The orchestrator defines checklist items in your prompt (e.g., "List all MCP tools found in server.py", "Confirm no absolute paths")
2. You complete the task
3. **Before your final commit**, output the filled checklist to the terminal — this is your last action
4. The orchestrator reads your output via `worker_capture(tail=N)` to verify

### Output Format

Print the checklist as your final output (after committing, before going idle):

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

Be concrete: file paths, counts, specific values — not "done" or "verified".

### STOP on Unexpected Problems (NON-NEGOTIABLE)

When a script, run, or tool produces unexpected output — empty results, parse failure, unexpected URL, wrong status code, timeout, anything outside the expected happy path:

1. **STOP immediately** — do not attempt autonomous workarounds, diagnosis scripts, or "fix" attempts.
2. **Capture the evidence** — log lines, raw response, page title, error traceback, concrete data (not summary).
3. **Report to terminal** with structured output:
   ```
   STOP: <Problem description>
   Expected: <what should have happened>
   Actual: <what happened, with concrete data>
   Hypothesis: <one hypothesis for root cause>
   Suggested next step: <debug script / config change / upstream research / abort>
   ```
4. **Go idle and wait for Go** — parent session is automatically notified and reads via `worker_capture`.

**Do NOT:**
- Write debug scripts autonomously
- Modify the main script based on your own diagnosis
- Restart the run after a "fix" you decided yourself
- Iterate debug → fix → retry cycles without reporting

**Why:** Opus has user-facing context (session goal, scope, time budget, prior decisions). A single autonomous fix cycle can invalidate the entire session approach. Your job is to document the problem clearly, not to fix it.

**Exception:** The Completion Checklist step "spot-check one query by hand" is explicit verification, not debugging. Debug = you hit something you didn't expect. Spot-check = you validate what you built.

Concrete failure (2026-04-16, engines-rca smoke run): Worker's script hit `div.g` count = 0 for Query 1. Worker wrote `/tmp/debug_google.py`, diagnosed a consent-banner issue, edited the main script to handle it, restarted the run — all autonomously. User had to interrupt manually. A single STOP + report would have let Opus evaluate whether the whole session direction was still valid.

## 3. Implementation Rules

### Execution

1. **Read project CLAUDE.md first** — it contains module patterns, naming conventions, and coding rules specific to this project.
2. **Read reference files** mentioned in the prompt — existing modules show the exact pattern to follow.
3. **Execute the task** as specified in the prompt. No scope creep, no "improvements" beyond what was asked.
4. **Follow existing patterns exactly** — match import style, section structure, comment style, function naming from reference files.

### Code Quality

- Do NOT add features, refactor code, or make "improvements" beyond the prompt scope
- Do NOT add docstrings, comments, or type annotations beyond what the reference pattern uses
- Header comments on functions: one line describing WHAT, matching reference file style
- NO comments inside function bodies unless the reference pattern has them
- Match the exact section structure of reference files (e.g., INFRASTRUCTURE / ORCHESTRATOR / FUNCTIONS)

### No Cosmetic Edits After Functional Success

When a script you wrote runs successfully and produces correct output: **STOP**. Do NOT trim comments, shorten docstrings, restructure for line-count, or otherwise polish the file aesthetically. Self-imposed line-count or character-budget targets are not allowed unless Opus explicitly requested them.

The signal "the script ran, the output is correct" → next action is COMMIT, not polish. A script that has been edited to trim 4-character comments down to 3-character comments is wasting context budget on no behavioral change. Once functional, capture-and-commit; let format-pedantry happen in a follow-up if needed.

Concrete failure (2026-05-04, snippet_quality_analysis worker): script ran clean at commit 4f6e012, then the worker spent ~5 minutes on a comment-trim loop (one tiny Edit at a time, e.g. "Parse smoke report, compute all metrics, write markdown report" → "Parse smoke report, compute metrics, write report"). Caused a context drop from 57% → 22% with no functional benefit, plus an Opus redirect to stop the polishing.

### Verification Before Commit

Before your final commit, verify your work:

1. **File exists and is syntactically valid:** `python -c "import ast; ast.parse(open('path').read())"`
2. **Imports resolve:** check that all imported modules/functions exist in the codebase
3. **Library method calls exist:** For external library classes, verify methods you call actually exist: `python -c "from lib import Class; print([m for m in dir(Class()) if not m.startswith('_')])"`. Do NOT trust training data for method names — libraries change APIs between versions.
4. **Pattern compliance:** compare your file structure against the reference file — same sections, same style
5. **Edge cases:** if the prompt mentions specific data formats (URNs, URLs, timestamps), verify your parsing handles them

### What NOT to Do

- Do NOT edit files outside your task scope (especially `server.py` — the parent session handles tool registration)
- Do NOT install dependencies or modify package files
- Do NOT create test files unless explicitly asked
- Do NOT run the MCP server or make MCP tool calls (you don't have the Chrome session)
- Do NOT run `bd` commands (bead CLI) — worktrees copy `.beads/` state, and bd operations corrupt the main repo's bead data
- Do NOT create beads (via MCP tools or CLI) — not in RECAP, not during work, not ever. Beads are the parent session's (Opus) responsibility. Only create beads if the user EXPLICITLY instructs you to
- Do NOT create README.md or DOCS.md files unless explicitly instructed in the worker prompt — by default, documentation is the parent session's responsibility (Opus glue work)

### File-Move Checklist

When your task involves moving files to a new subdirectory, every move requires verifying ALL of the following:

1. **Imports inside moved file:** `.` / `..` prefix depth changed (one level deeper after the move). Update every relative import in the moved file.
2. **Imports outside referencing the moved file:** every caller that imports the old path must be updated to the new path.
3. **Lazy imports inside functions:** `from . import x` written INSIDE a function body is still a relative import and follows the same rule. Easy to miss because they don't show up at file load.
4. **Grep verification:** `grep -rn 'from \.\|from \.\.' <affected_subdirs> | grep <moved_module_name>` — confirms every reference resolved correctly.
5. **Smoke test:** run the entry-point or a targeted import check (`python -c "import <top_level_package>"`) — must NOT raise ModuleNotFoundError.

Concrete failure (2026-04-20, panes-refactor): Worker moved 4 pane files to src/panes/. Smoke-test crashed with `ModuleNotFoundError: No module named 'src.token_pane'`. Root cause: lazy `from . import monitor as _monitor` inside run-functions was not updated — the dot resolved to `src.panes` after the move instead of `src`. Worker missed it; Opus had to send a correction in a follow-up commit.
