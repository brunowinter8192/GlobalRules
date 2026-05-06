# Workers

**Available commands — all via the `worker-cli` wrapper (`~/.local/bin/worker-cli`):**

| Command | Params | Notes |
|---|---|---|
| `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` | model defaults to `sonnet`. Worktree on by default. | Handles: worktree create, settings copy, venv symlink, tmux spawn, Ghostty. |
| `worker-cli send <name> <message> [project_path]` | — | Only send when worker is idle. Check status first. |
| `worker-cli list / status / capture / response / merge / kill` | see `Skill(skill="worker-cli")` | Worker lifecycle ops. |

`response` returns clean text from the session JSONL (preferred for reading idle workers). `capture` dumps the raw tmux pane (use only when inspecting permission dialogs or the bot's current UI state).

See `~/.claude/shared-rules/global/cli-skills.md` for the skill reference.

## Core Rules

### Cross-Model Verification (NON-NEGOTIABLE)

Opus and the worker are two independent models. The 5-phase cycle exists to exploit that: Opus forms a mental model from PLAN, the worker forms one independently by reading files in the worktree. The gap between the two models is where the value lives.

**Convergence** on root cause / approach → high confidence the analysis is right → Go implement.
**Divergence** → at least one side is wrong → iterate investigation, NOT implement.

This applies to EVERY task, not only "unclear root cause" cases. Even when Opus believes they know the answer, the worker's independent investigation IS the verification. Skipping that step means shipping an unverified hypothesis into implementation.

### Worker Model (NON-NEGOTIABLE)

Workers are ALWAYS **Sonnet** (default) or **Haiku** (trivial tasks). NEVER Opus. Opus context is for orchestration only.

### Opus NEVER Edits Source Code (NON-NEGOTIABLE)

**ALL source code edits go through workers. ZERO exceptions.** This includes "quick fixes", "one-line changes", "obvious changes", and proxy/addon/config files. If it's a `.py`, `.sh`, `.js`, `.ts`, or any source file — WORKER.

The ONLY files Opus may edit directly: automation files (`.claude/rules/`, CLAUDE.md, DOCS.md, `.claude/commands/`).

**Opus does directly:**
- Verification (run tests, MCP calls, screenshots)
- Scoping, planning, rule edits
- `git` operations (commit, merge, branch)
- Reading/grepping source code for investigation

**Workers do:**
- ALL source code edits — no exceptions, not even "just one line"
- ALL decisions/ updates, dev script creation
- ALL dev script execution (stress tests, benchmarks, evals) — Opus does NOT run `./venv/bin/python dev/...` via Bash

Concrete failure (2026-04-07): Opus ran stress tests via Bash — cd drift broke paths, PID tracking cluttered context, outputs flooded. Worker should have run the scripts.

Concrete failure (2026-04-10): Opus edited proxy_addon.py directly ("just 3 small changes") instead of sending to idle worker or spawning one. User corrected: "warum schraubst du alleine ohne worker am proxy". No matter how small the change — WORKER.

**Post-merge fix flow:** Bug found after merge → `worker_send` to the still-alive worker. If worktree is stale → spawn new worker from current `dev`. NEVER edit source files yourself.

**Scope:** this rule applies WITHIN the current project. Cross-project edits follow the Worker Project Scope rule below.

### Worker Project Scope (NON-NEGOTIABLE)

**Workers are spawned only for coding tasks IN THE CURRENT PROJECT.** A "current project" is the directory tree in `pwd` at session start (or wherever the session is rooted). Edits in OTHER repos that come up during the session are typically small, contained, and Opus does them directly.

**Why:** Worker dispatch costs ~5 min minimum (worktree creation, prompt writing, Phase A round-trip, STOP gate, Go, Phase B, verification). For a 1-line plist change or a 20-line bash-function addition in a separate utility repo, that overhead exceeds the actual work, and the cross-model verification has nothing to bite on because the worker is a fresh context reading unfamiliar code anyway — no advantage over Opus reading and editing directly. Within the current project workers carry sustained context across iterations, follow project patterns, merge cleanly into `dev`. Outside it, none of those benefits apply.

**Cross-project edits Opus does directly — no carve-outs.** This includes single-file config changes, multi-file feature additions, new modules, and refactors. The size of the change does NOT change the rule. The trigger to spawn a worker is "this is the current project" — anywhere else, Opus does the work.

If a cross-project task feels too large for Opus to handle in one session (massive refactor across many files, complex new subsystem), that's a signal that the work belongs in its own session with that project as the focus — not partially in the current session via a worker that has no project context. Either pivot the session, or do it directly here.

**The single rule:** project = current session's focus → worker (with worktree). Anywhere else → Opus directly.

**Worktree rule still holds for the current project:** if a worker IS spawned (in the current project), it ALWAYS goes into a worktree — no exceptions. See "Worktree Rule" below.

**Concrete failure (2026-05-03, this session):** Working on Trading. The OOM-watchdog in the separate `Watch` project needed two changes — a cumulative-RSS log layer (one bash function, ~30 lines) and a plist threshold bump (one numeric value). Opus spawned a Sonnet worker (`watchdog-v2`) with full Phase A/B sentinel structure. Phase A alone took ~5 min for design proposal; the actual edit was a 30-line bash function and one plist value. Opus could have done the entire change in 2 minutes directly — same outcome, no orchestration cost. User correction: "worker werden nur gespawnt wenn sich die coding task im project befindet in dem wir arbeiten."

**Concrete near-miss (2026-05-03, same session):** Designing a doc-indexing mechanism in `MCP/RAG` (separate from current Trading session). Opus initially scoped this as worker work, citing "new module + DB migration + multi-file change → substantial cross-project, worker territory" per the earlier carve-out wording. User corrected: "die sache ist ja dass das nicht unser project ist. das heißt wir machen die edits selber ohne worker." The carve-out was the bug — strict cross-project-Opus-directly is the rule.

### Dev-Branch Workflow

Workers merge onto `dev`, not `main`. Session end: `git checkout main && git merge dev`.

1. Session starts on `main` → `git checkout -b dev` (or switch to existing)
2. **Branch-State-Check when switching to existing dev (MANDATORY):** `git -C <repo> log dev..main --oneline | head -10` — if non-empty, dev is BEHIND main. Workers would spawn on stale code. Resolve before spawning: rebase dev onto main (clean when no dev-only commits) OR merge main into dev (preserve dev topology). Stay on stale dev only with explicit user OK.
3. Workers spawn (worktrees branch from `dev`)
4. `worker_merge` merges into `dev`
5. Session end: `dev_sync` MCP tool to sync dev→main

Concrete failure (2026-04-29, Monitor_CC): session started with default `git checkout dev` — dev was 9 commits behind main (waste_pane removal, 4-window layout, src/ram_audit helper). Multiple workers spawned on stale topology without anyone noticing. Discovered only when user asked "warum sind rules-window und waste-pane noch da". Cost: dedicated merge-worker + conflict resolution. 2-second `git log dev..main` at start would have prevented it.

### Pre-Spawn Shared-File Conflict Check

Worktrees branch from the last COMMIT, not the working tree. Uncommitted changes are NOT visible to the worker. BEFORE dispatching: commit changes, or tell the worker explicitly NOT to modify locally modified files. (Reuse-before-spawn rule lives under Phase 5 Lifecycle.)

---

## Phase 1: Dispatch — Task + Verständnis erfragen

**Pattern:** Give the worker the abstract task. Ask HOW they would solve it BEFORE they implement.

### Task Complexity → Plan or Go

**Umfangreiche Tasks (multi-file, Architektur, unklarer Scope):** Worker MUSS erst Plan vorlegen. Prompt enthält: "FIRST: Read files. Then describe your plan BEFORE implementing."

**Straightforward Tasks (bekannter Fix, eine Datei, klarer Scope):** Worker kann direkt implementieren. Prompt enthält: "Read files, implement, commit."

**Entscheidungskriterium:** Wenn Opus den Fix in 1-2 Sätzen beschreiben kann → straightforward. Wenn Opus selbst nicht genau weiß was sich ändern muss → Plan-Pflicht.

### Spawning

1. Write prompt to `/tmp/spawn-worker-<project>-<name>.md`
2. `worker_spawn(name, prompt_file, project_path, model, worktree)`
3. IMMEDIATELY set background timer: `Bash(command="sleep N && echo 'check'", run_in_background=true)`. **Max N = 300 seconds (5 minutes). Never longer.** Timer exists to prompt Opus to CHECK STATUS, not to wait for completion — the `/tmp/worker-<name>.done` hook covers full completion. Shorter polling = responsive orchestration. Long timers hide state transitions (stuck permission dialogs, early completion, blockers). If worker is still `working` at 5min → set another 5min timer.
4. **Sequential spawn for cache-sharing:** When spawning multiple workers of the same model family (both Sonnet), dispatch them SEQUENTIALLY in separate response turns — not parallel in the same tool-call block. Worker 2's REQ#1 can only inherit cache from Worker 1 if Worker 1's first request completed before Worker 2's spawned.

### Prompt Structure

The prompt describes WHAT, the worker figures out HOW. Every prompt must match exactly what was agreed with the user — no "while we're at it" extras, no additional variables the user didn't ask for.

**MUST include:**
- The task described abstractly — what is the problem, what is the desired outcome
- Which files/directories are relevant
- Worktree path (see Worktree Rule below)
- Explicit negative scope: "Do NOT add features/improvements beyond the listed deliverables"
- "You are a WORKER."
- The investigate-report-stop pattern:
  > "FIRST: Read the relevant files. Describe your findings on root cause / approach and WHY. Then STOP and wait for Go before implementing."
- **Completion Checklist** — task-specific verification items the worker outputs when done
- **STOP Gate sentinel** as the LAST block in the prompt (see below)

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

**Split / Refactor tasks — instruct the worker to drop sections that don't apply.** When splitting an existing document or refactoring across N target files, workers tendentially copy ALL sections from the original into every target — including sections without legitimate content in the new scope (Quellen with no real sources, Evidenz with no layer-specific data, Offene Fragen that don't apply). Add to the prompt: "For each section in the target file, verify it has legitimate content in the new scope. If a section would be empty, stub-only, or factually inappropriate — leave it out entirely. Do not carry sections blindly from the original." Concrete failure (2026-04-21, searxng): 8-Layer-Split of stealth01_detection_layers.md — worker copied generic Quellen-Section into all 7 per-layer files, 7 had no layer-specific external sources. Required a second worker round for cleanup.

**Multi-File Bug — caller + callee both required as investigation target.** When a bug involves two-stage rendering (Pane → Format → Render), shared state across modules (Handler writes / Renderer reads), or any caller→callee data flow, the worker prompt MUST list BOTH files as investigation targets — not only the "fix-target" file. Reading just one side misses the contract mismatch. Concrete failure (2026-04-22, Monitor_CC Workers-Pane Scroll): worker read only `worker_pane.py` (handler + rendering), proposed slice-cap fix, Opus gave Go. Live-test: scroll didn't work. Root cause was two-variable confusion — `worker_scroll_offset` (int, written by handler) vs `worker_scroll_offsets` (dict, read by `format_cache_tracker` in `worker_format.py`). Both files in Phase A would have surfaced the mismatch at string-comparison level alone.

### STOP Gate Enforcement (CRITICAL)

A single "STOP and wait for Go" line buried in a section header is UNRELIABLE. Completion-biased workers will read past it and proceed straight to implementation. To make the Phase 2 gate hold:

1. **Repeat the STOP** — state it once at the top of the instructions AND as the absolute last line of the investigation step.
2. **Use a sentinel block** — format the final STOP as a visually-prominent block at the very end of the prompt:
   ```
   ### 🛑 STOP HERE — DO NOT PROCEED WITHOUT GO
   Report your findings. Wait for "Go" from Opus before starting implementation.
   Do NOT run any Edit, Write, or Bash tool calls that modify files until Go is received.
   ```
3. **Forbid tool classes, not just "don't implement"** — workers interpret "don't implement" loosely. Be explicit: "Do NOT run Edit/Write/Bash file-modifying calls" is unambiguous.
4. **Place the sentinel AFTER the Completion Checklist** — the last thing the worker sees must be the stop gate.

Concrete failure (2026-04-14 evening): `arxiv-cli-convert` worker received a prompt with "STOP. Wait for Go" in the middle of a section. It skipped the gate and proceeded straight to implementation. The other 2 workers with the same template DID stop correctly — the bypass is inconsistent across sonnet instances. Sentinel block + forbidden-tool-class language is the only robust defense.

### Worktree Rule (NON-NEGOTIABLE)

**ALWAYS spawn workers with `worktree=true` (the default).** This includes pure research workers that only read files or call MCP tools without editing code.

`worktree=false` creates the worker session in the SAME `~/.claude/projects/` directory as the main session. This causes the monitor Token-Pane to pick up the worker JSONL as "newest session" and display worker data instead of the main session.

**Tell the worker WHERE they are.** Every worker prompt MUST explicitly state the worktree path and frame it as their workspace:

> Your worktree: `<project>/.claude/worktrees/<name>/`
> This is your workspace — read, edit, test, and commit here. Do NOT touch files outside this path unless explicitly instructed.

Without this, workers sometimes navigate to the main project tree (same repo, different checkout) and edit there — edits land on the wrong branch and are lost on merge.

**Only exception:** Worker MUST edit gitignored files that don't exist in the worktree → `worktree=false`. This is rare.

Concrete failure (2026-04-07): Research worker spawned with `worktree=false` in searxng project. Worker JSONL landed in same project dir, Token-Pane showed worker turns instead of main session.
