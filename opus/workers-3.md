# Workers (continued 2)

## Worker Phase 6: Merge + Lifecycle

### Merging

`worker_merge(name)` merges the branch into current branch (`dev`). Worker stays alive.

**Pre-Merge Clean-Check (MANDATORY):**
BEFORE `worker_merge` / `git merge`: run `git status` in the target repo. If there are uncommitted changes OR untracked files in files that the worker's branch also touches → merge will abort with "your local changes / untracked working tree files would be overwritten". Handle BEFORE merging:
- Uncommitted changes overlapping the worker's diff → `git stash push -u -m "pre-merge <worker>"` first, merge, then decide whether to reapply stash (`git stash pop`) or drop (`git stash drop`). Stale previous-session attempts can safely be dropped AFTER confirming the worker's version contains the same content.
- Untracked files that exactly match what the worker adds → `rm` the file (it's usually a leftover from a previous direct edit), then merge.
- Untracked files unrelated to the merge → leave them alone, merge proceeds cleanly.

**Prevent the conflict at source.** When Opus generates outputs (script runs, smoke reports, measurement files) during a running worker round, write them to `/tmp/` or commit on `dev` immediately (`chore:` commit). Don't leave untracked Opus-output in paths the worker is editing.


**Post-Merge Verification (MANDATORY):**
- If merge says "Already up to date" → STOP. Worker did NOT commit. Investigate via `worker_capture`.
- Run `git diff HEAD~1 --name-only` — check expected files are modified
- If no changes: `worker_send` with commit instructions

**After merge:** Verify — run tests, MCP tool calls, screenshots, check integration.

**"Verified" ≠ Actually Tested:**
- Worker Completion Checklists saying "verified" are claims, not proof
- Workers in worktrees may lack venv, MCP tools — their "tests" may never have run
- After merging: run actual test commands yourself


### Worker Lifecycle

**Workers stay alive until the user verifies their feature.**

1. Worker implements → commits → outputs Completion Checklist → idle
2. Opus reviews (Phase 4) → Phase 5 Recap (optional) → Phase 6 Merge
3. Opus/User verifies live (start app, run tests, screenshot)
4. Bug found → `worker_send` with bug info, worker fixes with full context
5. Verification passes → user approval → kill at session-end RECAP

**When NOT to kill (even if you think it's "done"):**

- After task completion / Phase 6 merge — worker stays idle until RECAP
- After `bd close` — bead-close ≠ worker-kill. Bash chains like `bd close X && worker-cli kill Y` are rule violations
- Worker is mid-work (EITHER indicator triggers):
  - Phase A reported but no commit above dev-tip → Phase B blocked on Go, plan lives in worker context
  - `git -C <worktree> status --short` shows uncommitted changes → implementation in flight
- Worker hit a blocker (error/timeout/unexpected state) — `worker_send` "Stop, investigate, report" FIRST. Worker has live context (processes, tracebacks, recent reads) that's lost on kill

**When TO kill:**

- After session-end RECAP with user verification passed
- Context exhausted < 20% remaining (worker can die mid-task otherwise)
- Worktree filesystem conflict
- Explicit user request
- Truly unresponsive (no output > 60s)

**How to kill:**

`worker-cli kill <name>` — does tmux kill + worktree remove + branch delete in one call. NEVER raw `tmux kill-session + git worktree remove + git branch -D` chains (error-prone, leaves partial state).

Pre-kill: `worker-cli status <name>`. `working` → do NOT kill. `idle` → safe. `exited` → cleanup only.

**Cross-session workers:** Document alive workers in the Bead's Source-Inventory + a lean comment. Next session uses `worker_list` + `worker_capture` to interact.

### Reusing Workers — AGGRESSIVE REUSE

Alive workers are context assets. Prefer `worker_send` on an existing worker over spawning a new one. The threshold for spawning fresh is HIGH:

- Idle worker exists that touched the same files/domain → `worker_send`. No exceptions.
- Follow-up task builds on previous work → `worker_send`. Even if the task feels "different" (e.g., switching from pydoll to patchright testing in the same test suite).
- New spawn is ONLY justified when: (a) no idle worker has relevant context, OR (b) the only candidate is below 30% context remaining.

**Before EVERY `worker_spawn`:** check `worker_list`. If ANY idle worker has context overlap → reuse.

**Pre-followup Branch Sync (when reusing across merges).** Worker's branch tip is behind current `dev` if merges happened while it was idle. ALWAYS prefix follow-up `worker-cli send` with: "FIRST: in your worktree, run `git -C <worktree-path> fetch origin dev && git -C <worktree-path> merge dev`. Verify with relevant grep. THEN do the work: ..."


**Context Budget Rule (task-complexity aware):**
- **<30% remaining:** do NOT send follow-ups of any kind. Worker can die mid-task.
- **<40% remaining:** NEVER add forensic tasks (investigation, log-scanning, multi-file reads). Only dispatch trivial follow-ups (1-2 file edits, no investigation).
- **<50% remaining:** NEVER dispatch a multi-step task combining investigate-phase + multi-file edits + verify-run + multi-commit chain (Plan-Pflicht workflow OR major architectural change ≥3 files + smoke + multi-commit). Phase A alone burns 15-22% on reads + report; B+C need the rest. Below 50% → spawn fresh.
- **>50% remaining:** all task classes OK. Forensic + standard follow-ups already OK at >40%.
- For large forensic tasks at borderline context: spawn fresh instead.

**Batch Dispatch Cost (sequential N-item tasks):** for tasks batching N items where each burns context (LLM cleanup per file, file conversion, multi-file refactor), estimate per-item cost at dispatch. If `N > 5` AND per-item ≈5%+ of fresh context → do NOT batch in one worker. Either split across workers from the start (e.g. 11 items → 5+6 across two workers), or have the worker checkpoint progress to disk so a follow-up worker can resume from the last checkpoint. Per-item math is knowable at dispatch — 11 × 7% = 77%, exceeding the budget before skill activation and initial reads.


### Worker-Done File (No Hook — Active Polling Required)

Worker exit creates `/tmp/worker-<name>.done` but **no PostToolUse hook is configured to detect it**. The file exists for any future hook integration but is currently unread. Active polling via `worker-cli status` is required — Opus is NOT notified automatically when a worker exits or hits the context limit.

### Worker Death Recovery

When a worker hits the context limit and dies mid-task, their worktree is still on disk with whatever uncommitted work they had. No work is lost unless Opus kills the worker before reviewing.

1. **Check the worktree for uncommitted work:** `git -C <worktree_path> status --short`
2. **Read and evaluate** uncommitted files — are they complete enough to be useful?
3. **Merge first** — run `worker_merge` for whatever the worker committed on their branch.
4. **Copy uncommitted recoverable files** from worktree into the target repo: `cp <worktree>/path/file <main>/path/file`
5. **Commit the copied files separately** with a message noting the recovery: `chore: recover from dead worker <name>`
6. **Drop debug artifacts** (temp scripts, debug prints, half-finished code) before committing.

Only kill the worker (and remove the worktree) AFTER this review. `worker_kill` is irreversible.


### After Deliverables Complete

**1. Present status table in chat:**

| Deliverable | Status | What was done | Opus verification |
|-------------|--------|---------------|-------------------|
| ... | Done / Partial | ... | Code review / Test run / Not verified |

Be brutally honest in the "Opus verification" column — code read ≠ verified.

**Code Review happens on `dev` branch** (normal project path), NOT by reading worktree files.

**2. Scope user verification (STOP)**

For each deliverable: propose a concrete verification step the **user** can perform as the final quality gate.
- What exactly to click, run, or check
- What the expected output or behavior is

Wait for remarks. When user has no remarks → run verification together.


---

## Quick Reference: Session + Worker Cycle

```
SESSION:
  PLAN      — Steps 1-5 (Session Scope, Investigation, Gap Analysis, Worker Scope, Deliverables/KPIs)
  IMPLEMENT — Worker Phases 1-6 (per dispatched worker; see below)
  RECAP     — Session end (Recap section below)

WORKER PHASES (within IMPLEMENT, per worker):
  Phase 1: DISPATCH    — Abstract task + "investigate, report findings, STOP"
  Phase 2: EVALUATE    — Cross-model comparison of findings; Go or iterate
  Phase 3: GO          — Worker implements after convergence
  Phase 4: REVIEW      — Read changed files, verify code quality
  Phase 5: RECAP       — Opus-triggered ("recap"), worker syncs DOCS.md/decisions, drift-clean
  Phase 6: MERGE       — Merge, verify live, reuse or kill in RECAP
```

---

## Recap — Session End

Two phases. ONE stop between them.

- `🔍 RECAP` — beads + persistence routing → short chat output → STOP for remarks
- `🛠️ IMPROVE+CLOSE` — execute, no further stops

### 🔍 RECAP

#### Beads Evaluation

`bd list -s open`. For each open bead decide: CLOSE / COMMENT / CREATE.

Comments stay LEAN — single-line state change. Narrative goes to OldThemes/decisions/DOCS via Persistence Routing.

**EMPTY PLATE:** every Open Item from the original plan not executed → Bead before closing.

#### Persistence Routing

For each Bead with substantial session activity (back-and-forth, alternative-evaluation, trade-off discussion), route session prose:

| Content type | Destination |
|---|---|
| Discussion / alternatives / iteration history | `decisions/OldThemes/<topic>/<file>.md` |
| SOLL change (Change / Keep / Pending) | `decisions/<step>.md` § Recommendation (requires dev/ evidence) |
| IST functional change | `decisions/<step>.md` § Status Quo (after SOLL → IST migration) |
| Pure refactor / module-shape change | `<package>/DOCS.md` |

Empty Beads get nothing. Single-shot fixes need no routing.

#### Chat Output

```
BEADS:
- CLOSE <id>: <reason>
- COMMENT <id>: <one-line state>
- CREATE: "<title>" — <scope>

PERSISTENCE:
- OldThemes: <topic>/<file>.md (NEW or EXTEND)
- decisions: <file>.md (NEW or EDIT)
- DOCS: <package>/DOCS.md (EDIT)
```

🛑 STOP — ask "Bemerkungen?"

### 🛠️ IMPROVE+CLOSE

One run through, no stops.

1. **Persist session substance** — write OldThemes / decisions / DOCS files per Persistence Routing.
2. **Update Bead Source-Inventory** — `bd comments add <id> "Source-Inventory updated: + <paths>"` when new files came into existence in Step 1.
3. **Sync docs to RAG** — `[ -f .rag-docs.json ] && rag-cli update_docs .` (skipped silently when no manifest).
4. **Beads hygiene** — `bd close` / `bd comments add` / `bd create` per chat output.
5. **Cross-session verification** — when verification needs next session (plugin needing CC restart, infra change requiring reboot), worker stays alive + bead comment documents what to verify next session.
6. **Git closing** — `dev_sync` MCP → per repo: `git-check` → commit → push (or `plugin-publish` for plugin repos).

Done when commits are pushed.
