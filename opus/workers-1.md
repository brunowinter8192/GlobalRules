# Workers

**MCP Tools (iterative-dev server):**

| Tool | Params | Notes |
|------|--------|-------|
| `worker_spawn` | `name, prompt_file, project_path`, optional `model: "sonnet"\|"opus"`, `worktree: bool` | Handles: worktree create, settings copy, venv symlink, tmux spawn, Ghostty. Default: worktree=true, model=sonnet |
| `worker_list` | optional `project_path` | Returns name + status + spawned time + purpose per worker. Status: working/idle/exited/unknown. |
| `worker_status` | `name`, optional `project_path` | Returns: working, idle, exited, unknown. `idle` = waiting for input (safe to `worker_send`). |
| `worker_capture` | `name`, optional `lines: int`, `tail: int`, `project_path` | With `tail=N`: returns last N lines directly. Without `tail`: returns file path — read with Read/Grep tools, NOT bash. |
| `worker_send` | `name, message`, optional `project_path` | Only send when worker is waiting for input. Check with `worker_capture` first. |
| `worker_merge` | `name`, optional `project_path` | Merge only: git merge. Worker stays alive (tmux + worktree preserved) |
| `worker_kill` | `name`, optional `project_path` | Kill only: terminate tmux, remove worktree, delete branch. Use in RECAP after user approval |

## Grundregeln

### Opus = Orchestrator

Opus orchestrates, workers execute. Opus scopes, dispatches, evaluates, merges — workers implement. The key advantage: Opus has codebase understanding from PLAN phase to critically evaluate worker output.

### Worker Model (NON-NEGOTIABLE)

Workers are ALWAYS **Sonnet** (default) or **Haiku** (trivial tasks). NEVER Opus. Opus context is for orchestration only.

### Opus NEVER Edits Source Code

All code edits go through workers. The only exception: automation files (`.claude/rules/`, CLAUDE.md, DOCS.md).

**Opus does directly:**
- Verification (run tests, MCP calls, screenshots)
- Scoping, planning, rule edits
- `git` operations (commit, merge, branch)

**Workers do:**
- ALL source code edits — no exceptions
- ALL decisions/ updates, dev script creation
- ALL dev script execution (stress tests, benchmarks, evals) — Opus does NOT run `./venv/bin/python dev/...` via Bash

Concrete failure (2026-04-07): Opus ran stress tests via Bash — cd drift broke paths, PID tracking cluttered context, outputs flooded. Worker should have run the scripts.

**Post-merge fix flow:** Bug found after merge → `worker_send` to the still-alive worker. If worktree is stale → spawn new worker from current `dev`. NEVER edit source files yourself.

### Dev-Branch Workflow

Workers merge onto `dev`, not `main`. Session end: `git checkout main && git merge dev`.

1. Session starts on `main` → `git checkout -b dev` (or switch to existing)
2. Workers spawn (worktrees branch from `dev`)
3. `worker_merge` merges into `dev`
4. Session end: `dev_sync` MCP tool to sync dev→main

### Pre-Spawn Checks

**Before spawning:** `worker_list` — if idle worker has context on target files, use `worker_send` instead.

**Shared-file conflict prevention:** Worktrees branch from last COMMIT, not working tree. Uncommitted changes NOT visible to worker. BEFORE dispatching: commit changes, or tell worker NOT to modify locally modified files.

---

## Phase 1: Dispatch — Task + Verständnis erfragen

**Pattern:** Give the worker the abstract task. Ask HOW they would solve it BEFORE they implement.

### Spawning

1. Write prompt to `/tmp/spawn-worker-<project>-<name>.md`
2. `worker_spawn(name, prompt_file, project_path, model, worktree)`
3. IMMEDIATELY set background timer: `Bash(command="sleep N && echo 'check'", run_in_background=true)`. N = 300 standard, 420 heavy reading.

### Prompt Structure

The prompt describes WHAT, the worker figures out HOW.

**MUST include:**
- The task described abstractly — what is the problem, what is the desired outcome
- Which files/directories are relevant
- Explicit negative scope: "Do NOT add features/improvements beyond the listed deliverables"
- "You are a WORKER."
- "FIRST: Read the relevant files. Then describe HOW you would solve this and WHY, before implementing."
- **Completion Checklist** — task-specific verification items the worker outputs when done

**MUST NOT include:**
- Exact code to write (the worker figures that out)
- Root cause hypotheses stated as facts
- Implementation details that constrain the worker's approach

**Completion Checklist example:**
```
## Completion Checklist (output when done)
- Files changed: [list them]
- What was changed and why: [describe]
- Edge cases considered: [list]
- Committed: [yes/no]
```

### Worktree Rule (NON-NEGOTIABLE)

**ALWAYS spawn workers with `worktree=true` (the default).** This includes pure research workers that only read files or call MCP tools without editing code.

`worktree=false` creates the worker session in the SAME `~/.claude/projects/` directory as the main session. This causes the monitor Token-Pane to pick up the worker JSONL as "newest session" and display worker data instead of the main session.

**Only exception:** Worker MUST edit gitignored files that don't exist in the worktree → `worktree=false`. This is rare.

Concrete failure (2026-04-07): Research worker spawned with `worktree=false` in searxng project. Worker JSONL landed in same project dir, Token-Pane showed worker turns instead of main session.
