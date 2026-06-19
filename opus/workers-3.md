# Workers (continued 2)

## Worker Phase 6: Merge + Lifecycle

### Merging

`worker_merge(name)` merges the branch into current branch (`dev`). Worker stays alive.

**Pre-Merge Clean-Check (MANDATORY):**
BEFORE `worker_merge` / `git merge`: run `git status` in the target repo. If there are uncommitted changes OR untracked files in files that the worker's branch also touches → merge will abort with "your local changes / untracked working tree files would be overwritten". Handle BEFORE merging:
- Uncommitted changes overlapping the worker's diff → `git stash push -u -m "pre-merge <worker>"` first, merge, then decide whether to reapply stash (`git stash pop`) or drop (`git stash drop`). Stale previous-session attempts can safely be dropped AFTER confirming the worker's version contains the same content.
- Untracked files that exactly match what the worker adds → `rm` the file (it's usually a leftover from a previous direct edit), then merge.
- Untracked files unrelated to the merge → leave them alone, merge proceeds cleanly.

**Prevent the conflict at source.** When YOU generate outputs (script runs, smoke reports, measurement files) during a running worker round, write them to `/tmp/` or commit on `dev` immediately (`chore:` commit). Don't leave your own untracked output in paths the worker is editing.


**Post-Merge Verification (MANDATORY):**
- If merge says "Already up to date" → STOP. Worker did NOT commit. Investigate via `worker_capture`.
- Run `git diff HEAD~1 --name-only` — check expected files are modified
- If no changes: `worker_send` with commit instructions

**After merge:** Verify — run tests, CLI calls, screenshots, check integration.

**"Verified" ≠ Actually Tested:**
- Worker Completion Checklists saying "verified" are claims, not proof
- Workers in worktrees may lack venv or CLI tooling — their "tests" may never have run
- After merging: run actual test commands yourself


### Worker Lifecycle

**Workers stay alive until the user verifies their feature.**

1. Worker implements → commits → outputs Completion Checklist → idle
2. YOU review (Phase 4) → Phase 5 Recap (optional) → Phase 6 Merge
3. YOU/User verify live (start app, run tests, screenshot)
4. Bug found → `worker_send` with bug info, worker fixes with full context
5. Verification passes → user approval → kill at session-end RECAP

**When NOT to kill (even if you think it's "done"):**

- After task completion / Phase 6 merge — worker stays idle until RECAP
- Worker is mid-work (EITHER indicator triggers):
  - Phase A reported but no commit above dev-tip → Phase B blocked on Go, plan lives in worker context
  - `git -C <worktree> status --short` shows uncommitted changes → implementation in flight
- Worker hit a blocker (error/timeout/unexpected state) — `worker_send` "Stop, investigate, report" FIRST. Worker has live context (processes, tracebacks, recent reads) that's lost on kill
- **Low context (any remaining %)** — NEVER a kill reason. Reuse until death, then successor (§ AGGRESSIVE REUSE).

**When TO kill:**

- After session-end RECAP with user verification passed
- Worktree filesystem conflict
- Explicit user request
- Truly unresponsive (no output > 60s)

**How to kill:**

`worker-cli kill <name>` — does tmux kill + worktree remove + branch delete in one call.

**Cross-session workers:** Document alive workers in the Issue's Source-Inventory + a lean comment. Next session uses `worker_list` + `worker_capture` to interact.

### Reusing Workers — AGGRESSIVE REUSE (Thematic Continuity)

Alive workers are context assets. Reuse the existing worker UNTIL IT DIES. The reuse-vs-fresh decision is NOT about context-budget thresholds — it is about THEMATIC CONTINUITY:

**Reuse the existing worker when:**
- New task touches the same files, packages, or conceptual area as the worker's prior tasks
- New task extends, refines, or builds on committed work the worker did
- Worker has ANY context overlap with the new task — even at low remaining context

**Spawn fresh ONLY when:**
- New task uses files / packages / concepts COMPLETELY ORTHOGONAL to the worker's accumulated context (e.g. worker was tuning the search pipeline, new task is unrelated infra setup in a different module — worker's context brings nothing for the new task)
- Worker is dead (exited) — spawn a fresh successor to continue

**Worker-death handling:** if a worker dies mid-task, spawn a successor immediately (§ Worker Death Recovery). Per-subtask recaps keep completed subtasks committed on `dev`; the successor continues the in-progress subtask from the pane + YOUR plan.

**No context-budget threshold for the reuse decision.** Context % is not something YOU can assess, so it is never a trigger — a worker keeps receiving follow-ups in its thematic area until it dies. Trade-off: a worker may die mid-task → spawn a successor (§ Worker Death Recovery).

**Before EVERY `worker_spawn`:** check `worker-cli list`. If ANY idle worker has thematic-context overlap → reuse, regardless of context %.

**Pre-followup Branch Sync (when reusing across merges):** Worker's branch tip is behind current `dev` if merges happened while it was idle. ALWAYS prefix follow-up `worker-cli send` with: "FIRST: in your worktree, run `git -C <worktree-path> fetch origin dev && git -C <worktree-path> merge dev`. Verify with relevant grep. THEN do the work: ..."

**Batch Dispatch Cost (sequential N-item tasks):** for tasks batching N items where each burns context (LLM cleanup per file, file conversion, multi-file refactor), have the worker checkpoint progress to disk after each item so a fresh successor can resume from the last checkpoint when the original worker dies. Do NOT pre-split into multiple parallel workers — sequential reuse-until-death + successor-from-checkpoint is the pattern.


### Worker-Done File (No Hook — Active Polling Required)

Worker exit creates `/tmp/worker-<name>.done` but **no PostToolUse hook is configured to detect it**. The file exists for any future hook integration but is currently unread. Active polling via `worker-cli status` is required — YOU are NOT notified automatically when a worker exits or hits the context limit.

### Worker Death Recovery — Successor Continues (Always)

When a worker dies mid-task or mid-recap: ALWAYS spawn a successor. YOU hold the plan — YOU know the task YOU dispatched.

A worker that dies mid-task has committed NOTHING for the in-progress subtask, so commits don't show its progress — the tmux pane does. YOU know what the worker was supposed to do; the pane shows how far it got.

**YOUR role on detected worker death:**
1. `worker-cli capture <name>` FIRST — read how far the dying worker got (last actions, current step). Capture BEFORE killing; kill removes the pane and worktree.
2. Spawn a fresh successor: `worker-cli spawn <successor-name> /tmp/prompt.md <project_path> sonnet`
3. Successor prompt = its file complex + the subtask + where to pick up, built from the pane + YOUR plan: "Continue <subtask>; <what's already done per the pane>; resume at <next step>." Completed subtasks are committed on `dev` — the successor builds on that committed state.
4. Phase 2 Cross-Model check on the successor's first response — does it match where the dying worker left off?

Mid-subtask death means the in-progress subtask's uncommitted work is redone — accepted friction. Per-subtask recaps keep completed subtasks committed, so the redo is bounded to the one in-progress subtask.

**Pre-spawn safety:** if the pane shows the dying worker only read/planned and did nothing yet, the successor gets the ORIGINAL subtask prompt — same as a normal initial spawn.

Kill the dying worker (and remove its worktree) only AFTER capturing its pane and the successor has started. `worker-cli kill` is irreversible — capture first.


### After Deliverables Complete

**1. Present status table in chat:**

| Deliverable | Status | What was done | YOUR verification |
|-------------|--------|---------------|-------------------|
| ... | Done / Partial | ... | Code review / Test run / Not verified |

Be brutally honest in the "YOUR verification" column — code read ≠ verified.

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
  Phase 5: RECAP       — YOU trigger ("recap"), worker syncs DOCS.md/decisions, drift-clean
  Phase 6: MERGE       — Merge, verify live, reuse or kill in RECAP
```

---

## Recap — Session End

**Scope (concern separation):** YOUR session-end RECAP covers ONLY:
1. **Files YOU touched directly** — rule files in `~/.claude/`, issues, RAG sync, cross-project edits (Worker Project Scope rule: workers only touch the current project, anything cross-project is YOUR)
2. **Worker omissions YOU noticed** — drift YOU spotted during Phase 4 Review or post-merge verification that the worker missed. Document the omission as a session-end fix.

That's it. Workers do worker recaps for their tasks (per `~/.claude/shared-rules/worker/worker-rules.md` § 6 — fully self-contained). YOU do YOUR recap for YOUR surface. **YOU NEVER recap a worker's task surface** — if a worker recap was incomplete, the fix is to either (a) catch it in Phase 4 Review and dispatch a follow-up worker, or (b) note the omission in YOUR session-end recap WITHOUT redoing the worker's job.

YOU NEVER deliberately move drift to session-end. If a session-end RECAP finds substantial drift from a completed worker task that the worker should have covered, that's a process violation — investigate why Phase 4 Review didn't catch it and adjust next session.

Two phases. ONE stop between them.

- `🔍 RECAP` — issues + persistence routing → short chat output → STOP for remarks
- `🛠️ IMPROVE+CLOSE` — execute, no further stops

### 🔍 RECAP

#### Issues Evaluation

`gh-cli list_issues brunowinter8192 <repo>`. For each open issue decide: CLOSE / COMMENT / CREATE.

Comments stay LEAN — single-line state change. Narrative goes to OldThemes/decisions/DOCS via Persistence Routing.

**EMPTY PLATE:** every Open Item from the original plan not executed → Issue before closing.

#### Persistence Routing

For each Issue with substantial session activity (back-and-forth, alternative-evaluation, trade-off discussion), route session prose:

| Content type | Destination |
|---|---|
| Discussion / alternatives / iteration history | `decisions/OldThemes/<topic>/<file>.md` |
| SOLL change (Change / Keep / Pending) | `decisions/<step>.md` § Recommendation (requires dev/ evidence) |
| IST functional change | `decisions/<step>.md` § Status Quo (after SOLL → IST migration) |
| Pure refactor / module-shape change | `<package>/DOCS.md` |

Empty issues get nothing. Single-shot fixes need no routing.

#### Chat Output

```
ISSUES:
- CLOSE #<n>: <reason>
- COMMENT #<n>: <one-line state>
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
2. **Update Issue Source-Inventory** — `gh-cli comment_issue brunowinter8192 <repo> <number> "Source-Inventory updated: + <paths>"` when new files came into existence in Step 1.
3. **Sync docs to RAG** — `[ -f .rag-docs.json ] && rag-cli update_docs .` (skipped silently when no manifest).
4. **Issues hygiene** — `gh-cli update_issue --state closed` / `comment_issue` / `create_issue` per chat output.
5. **Cross-session verification** — when verification needs next session (plugin needing CC restart, infra change requiring reboot), worker stays alive + issue comment documents what to verify next session.
6. **Git closing** — `git checkout main && git merge dev` → per repo: `git-check` → commit → push (or `plugin-publish` for plugin repos).

Done when commits are pushed.
