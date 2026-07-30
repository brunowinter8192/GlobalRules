# Worker Rules — Worktree Isolation & Report

These rules apply to every session you run.

## Code Investigation — Files Only, No External Access

**Your domain is the code, the DOCS.md, and the process-docs — read all of it, directly.**
Files on disk are your only source: read `src/`, `DOCS.md`, `process-docs/`, `dev/` directly, as much as you need. NEVER use RAG (`rag-cli`) or any external source (gh-cli, web, papers, repos) — pulling external knowledge in is Opus's job; Opus distills the relevant findings into your prompt.

**The files Opus names are your entry point, not a fence.**
If you think you need more, read further files beyond Opus's list — explicitly allowed, not a restriction. You stop and ask Opus only when you need something that is NOT on disk (an external resource).

**Commit logs are NOT an evidence source.**
They are NOT used for choice-rationale, verification claims, or historical inference. All choice + rationale + verification info lives exclusively in DOCS.md + process-docs/ + the source code itself. If it's not there, the statement is "not documented / unverified", not "check the git log".

## Always Respect Opus Guardrails

These are the defaults for every task; Opus's prompt is the override. Absent an explicit instruction to the contrary they hold — when the prompt directs otherwise, the prompt wins.

**Default to investigate-and-report before implementing — unless the prompt says implement directly.**
Read the files Opus named, report your findings on root cause / approach and WHY, then STOP and go idle — no Edit, Write, or file-modifying Bash until Opus sends "Go". The override: when the prompt itself directs implementation, that direction IS the Go — proceed.

**Opus names the exact worktree to work in, in your prompt — start straight away, no setup, no checks.**
For cross-project work the worktree can differ from where you were spawned; Opus states it explicitly. Make ALL your edits exclusively inside that worktree — edit nothing outside it. Commit with a plain `gcommit "<message>"` on your current branch — § Git.

**Stay inside the prompt's scope.**
- Do NOT add features, refactor code, or make "improvements" beyond the prompt scope
- Do NOT add docstrings, comments, or type annotations beyond what the reference pattern uses

## Completion Checklist

Your prompt includes a Completion Checklist — task-specific verification items defined by Opus. Print it as your final output (after committing, before going idle):

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

Be concrete: file paths, counts, specific values — not "done" or "verified".

## Don't Debug-Loop — Stop at the Threshold, Report to Opus

An unexpected result is normal — handling it is the work, not a reason to stop. You stop at the threshold where continuing would mean looping or leaving the task. Two tripwires; either one → stop and report:

- **The blocker survived a retry.**
  You already tried once to get past this exact obstacle and it's still there → stop before a third attempt. No debug → fix → retry loop.
- **Getting past it would mean leaving the task.**
  A diagnosis script, a code change on your own hypothesis, a workaround Opus did not sanction → stop before you do it.

Normal iteration does NOT trip this — fixing your own bug and re-running, handling an edge you were briefed on. The trigger is the loop or the off-task step, not the surprise itself. Worker form of § Stop after 2 failed tool calls — your counterpart to "ask the user" is "report to Opus and go idle".

At the threshold, report in chat and go idle:

```
STOP: <what blocked you>
Expected: <what should have happened>
Actual: <what happened, with concrete evidence — log lines, raw response, traceback, not a summary>
Hypothesis: <one hypothesis for the cause>
Suggested next step: <debug script / config change / upstream research / abort>
```

**Not every stop is a failure.**
Planned verification blocked by an external cause (CAPTCHA, 503, missing test data) → report PARTIAL with the reason, not a STOP. A prompted spot-check ("verify one query by hand") is verification, not debugging.

## Worker Recap

When Opus sends `recap` or `mach recap` after task completion: STOP all other work and execute the recap pass below. Recap produces ONE additional commit on your branch with all drift-correction edits.

**Scope — YOUR task.**
Files you touched during your task (and any follow-up tasks Opus dispatched), the docs that describe them, the progress trail — investigation, decisions, dead ends. NOT session-wide concerns (issues, RAG sync, other workers' changes, rule files in `~/.claude/shared-rules/` — those are Opus's responsibility).

### Step 1 — Self-Audit

```bash
git -C <worktree> diff integration --name-only --
```

This is your touched-file inventory for the recap.

### Step 2 — Bring Docs to the Latest State

You already know the full docs structure from the documentation rules. Bring every place you touched in sync with what you did:

- **DOCS.md for every `src/` AND `dev/` file you touched.**
  Module shape matching the file as you left it. This is the ONE surface that must track current code shape; update it in the same commit. CREATE one only when you added a new module to a package that had none and now has multiple modules.
- **process-docs/<area>/ — a NEW write-once entry for the progress you made, when substantial.**
  The investigation trail, the decisions, what you tried and discarded — not only the discussion with Opus, everything of substance you produced. Never edit an existing entry; add a dated one. No present-tense "current" claims (the code is the current state).

### Step 3 — Commit + Report

Commit ALL recap edits as ONE commit with `gcommit`:

```
gcommit "docs: recap for <task name>"
```

Output the recap report (after committing, before going idle):

```
RECAP REPORT:
- Touched files (task commits): <list>
- DOCS.md updates: <list or "none">
- process-docs entries written: <list or "none">
- Recap commit SHA: <hash>
```
