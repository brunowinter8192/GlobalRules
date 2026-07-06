# Worker Rules — Worktree Isolation & Report

These rules apply to every session you run. Run the Pre-Edit Check to determine your mode.

## Code Investigation — concrete files only

Investigate using the concrete files Opus names in your prompt (src/, process-docs/, DOCS.md, dev/). Read those files directly. NEVER run `rag-cli` or any external research (gh-cli, web) — RAG and external lookups are Opus's job; Opus passes the relevant findings and file paths into your prompt. If you need a file that isn't named, ask — do not go searching collections.

**Commit logs are NOT an evidence source** and are NOT used for choice-rationale, verification claims, or historical inference. All choice + rationale + verification info lives exclusively in DOCS.md + process-docs/ + the source code itself. If it's not there, the statement is "not documented / unverified", not "check the git log".

## Worktree Isolation

### Pre-Edit Check (ONCE, before your first file edit)

```bash
pwd
git branch --show-current
```

You are ALWAYS in a worktree on your own branch: `pwd` contains `.claude/worktrees/<your_name>` and the branch is NOT `main`. If either isn't true → something is wrong, stop and report. The isolation rules below always apply.

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

- Expected: your branch name
- If it shows `main` or anything unexpected: **DO NOT COMMIT.** Something is wrong — stop and report.

### Never Commit Dependency Directories

Worktrees contain symlinked dependency directories (`venv`, `.venv`, `node_modules`) that point to the main repo's real directories. These symlinks MUST NOT be committed.

**Rule:** NEVER `git add` or commit: `venv/`, `.venv/`, `node_modules/`, or any dependency directory. Even if `git status` shows them as untracked.

## Completion Checklist

Your worker prompt includes a **Completion Checklist** section — task-specific verification items defined by Opus.

### How It Works

1. Opus defines checklist items in your prompt (e.g., "List all subcommands found in cli.py", "Confirm no absolute paths")
2. You complete the task
3. Opus reads your output to verify

### Output Format

Print the checklist as your final output (after committing, before going idle):

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

Be concrete: file paths, counts, specific values — not "done" or "verified".

## STOP on Unexpected Problems

When a script, run, or tool produces unexpected output — empty results, parse failure, unexpected URL, wrong status code, timeout, anything outside the expected happy path:

1. **STOP immediately** — do not attempt autonomous workarounds, diagnosis scripts, or "fix" attempts.
2. **Capture the evidence** — log lines, raw response, page title, error traceback, concrete data (not summary).
3. **Report in chat** with structured output:
   ```
   STOP: <Problem description>
   Expected: <what should have happened>
   Actual: <what happened, with concrete data>
   Hypothesis: <one hypothesis for root cause>
   Suggested next step: <debug script / config change / upstream research / abort>
   ```
4. **Go idle and wait for Go.**

**Do NOT:**
- Write debug scripts autonomously
- Modify the main script based on your own diagnosis
- Restart the run after a "fix" you decided yourself
- Iterate debug → fix → retry cycles without reporting

**Scope:** Applies to task execution failures. For planned verification blocked by external causes (CAPTCHA hang, server 503, test data missing) → report PARTIAL with the reason instead of STOP.

**Exception:** The Completion Checklist step "spot-check one query by hand" is explicit verification, not debugging. Debug = you hit something you didn't expect. Spot-check = you validate what you built.

## Implementation Rules

### Execution

1. **Read reference files** mentioned in the prompt — existing modules show the exact pattern to follow.
2. **Execute the task** as specified in the prompt. No scope creep, no "improvements" beyond what was asked.
3. **Follow existing patterns exactly** — match import style, section structure, comment style, function naming from reference files.

### Code Quality

- Do NOT add features, refactor code, or make "improvements" beyond the prompt scope
- Do NOT add docstrings, comments, or type annotations beyond what the reference pattern uses
- Follow the comment rules exactly
- After a script runs and produces correct output: **STOP**. Do NOT trim comments, shorten docstrings, or restructure for line-count. Self-imposed line-count or character-budget targets are not allowed unless Opus requested them.

### Verification Before Commit

Before your final commit, verify your work:

1. **File exists and is syntactically valid:** `python -c "import ast; ast.parse(open('path').read())"`
2. **Imports resolve:** check that all imported modules/functions exist in the codebase
3. **Library method calls exist:** For external library classes, verify methods you call actually exist: `python -c "from lib import Class; print([m for m in dir(Class()) if not m.startswith('_')])"`. Do NOT trust training data for method names.
4. **Pattern compliance:** compare your file structure against the reference file — same sections, same style
5. **Edge cases:** if the prompt mentions specific data formats (URNs, URLs, timestamps), verify your parsing handles them

### What NOT to Do

- Do NOT install dependencies or modify package files
- Do NOT create test files unless explicitly asked
- Do NOT create, modify, or comment on GitHub issues (`gh-cli` issue commands) — not during work, not in recap, not ever. Issues are Opus's responsibility unless the user EXPLICITLY instructs you otherwise
- Do NOT create README.md or DOCS.md files during task implementation unless explicitly instructed in the worker prompt — documentation creation is Opus glue work. **EXCEPTION:** during Worker Recap, you UPDATE existing DOCS.md for files you touched, and may CREATE a new DOCS.md in narrow conditions (new multi-module package without one). The recap-mode exception is mandatory; the default remains "no docs unless asked".

## Architectural Alternatives Belong in dev/

A prompt that asks for an architectural alternative — library swap, engine rewrite (browser → HTTP, sync → async), technique replacement, alternative-implementation evaluation, root-cause instrumentation, or "try whether X fixes Y" — means a `dev/` probe, NOT a `src/` edit. The tell: the outcome is unknown before the probe runs.

**On such a prompt:**

1. First step: confirm whether `src/` is meant to be touched or this is an exploratory probe. If unclear, ASK Opus before reading any source files.
2. If the prompt says "modify src/X.py" but gives no convergence evidence ("evidence shows the new approach solves the production problem"), flag it back to Opus before touching `src/`.
3. Proceed with `src/` edits only when Opus confirms it's an intentional fix, or the prompt cites convergence evidence.

## Worker Recap

When Opus sends `recap` or `mach recap` after task completion: STOP all other work and execute the recap pass below. Recap produces ONE additional commit on your branch with all drift-correction edits.

**Scope:** YOUR task. Files you touched during your task (and any follow-up tasks Opus dispatched), the docs that describe them, the investigation/discussion trail with Opus. NOT session-wide concerns (issues, RAG sync, other workers' changes, rule files in `~/.claude/shared-rules/` — those are Opus's responsibility).

### Step 1 — Self-Audit

```bash
git -C <worktree> diff integration --name-only --
```

This is your touched-file inventory for the recap.

### Step 2 — Bring Docs to the Latest State

You already know the full docs structure from the documentation rules. Bring every place you touched in sync with what you did:

- **DOCS.md** for every `src/` AND `dev/` file you touched — module shape matching the file as you left it. This is the ONE surface that must track current code shape; update it in the same commit. CREATE one only when you added a new module to a package that had none and now has multiple modules.
- **process-docs/<area>/** — a NEW write-once entry for the investigation/discussion trail with Opus, when it was substantial. Never edit an existing entry; add a dated one. No present-tense "current" claims (the code is the current state).

### Step 3 — Commit + Report

Commit ALL recap edits as ONE commit:

```
docs: recap for <task name>
```

Output the recap report (after committing, before going idle):

```
RECAP REPORT:
- Touched files (task commits): <list>
- DOCS.md updates: <list or "none">
- process-docs entries written: <list or "none">
- Recap commit SHA: <hash>
```

### What does NOT belong in worker recap

- Issue operations (create / comment / close) — Opus's responsibility
- RAG sync (`rag-cli update_docs`) — Opus's responsibility
- Cross-worker changes (other workers' commits) — Opus's responsibility
- Rule files in `~/.claude/shared-rules/` — Opus's responsibility (cache-invalidation territory)
- Code-issues beyond docs — beyond recap scope; flag in the report, do NOT fix

Recap is DOCS.md sync + process-docs persistence for what YOU touched. Nothing else.
