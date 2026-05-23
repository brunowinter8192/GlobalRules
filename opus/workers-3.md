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

### Reusing Workers — AGGRESSIVE REUSE (Thematic Continuity)

Alive workers are context assets. Reuse the existing worker UNTIL IT DIES. The reuse-vs-fresh decision is NOT about context-budget thresholds — it is about THEMATIC CONTINUITY:

**Reuse the existing worker when:**
- New task touches the same files, packages, or conceptual area as the worker's prior tasks
- New task extends, refines, or builds on committed work the worker did
- Worker has ANY context overlap with the new task — even at low remaining context

**Spawn fresh ONLY when:**
- New task uses files / packages / concepts COMPLETELY ORTHOGONAL to the worker's accumulated context (e.g. worker was tuning the search pipeline, new task is unrelated infra setup in a different module — worker's context brings nothing for the new task)
- Worker is dead (exited) — spawn a fresh successor to continue

**Worker-death handling:** if a worker dies mid-task or hits context-floor, spawn a fresh successor immediately. The successor reads from `dev` (committed state) and continues. Mandatory Phase 5 recap after every stage (workers-2.md) ensures committed state is always current — a dying worker leaves clean state for the successor. Using a worker until death is preferable to premature fresh-spawn: accumulated context has real value, and the death-event is mitigated by recap-discipline + clean-state inheritance.

**No context-budget threshold for the reuse decision.** Workers below 30% can still receive follow-ups in their thematic area. Trade-off: low-context worker may die mid-task → fresh successor inherits clean committed state and finishes. This is strictly better than spawning fresh prematurely and losing accumulated context.

**Before EVERY `worker_spawn`:** check `worker-cli list`. If ANY idle worker has thematic-context overlap → reuse, regardless of context %.

**Pre-followup Branch Sync (when reusing across merges):** Worker's branch tip is behind current `dev` if merges happened while it was idle. ALWAYS prefix follow-up `worker-cli send` with: "FIRST: in your worktree, run `git -C <worktree-path> fetch origin dev && git -C <worktree-path> merge dev`. Verify with relevant grep. THEN do the work: ..."

**Batch Dispatch Cost (sequential N-item tasks):** for tasks batching N items where each burns context (LLM cleanup per file, file conversion, multi-file refactor), have the worker checkpoint progress to disk after each item so a fresh successor can resume from the last checkpoint when the original worker dies. Do NOT pre-split into multiple parallel workers — sequential reuse-until-death + successor-from-checkpoint is the pattern.


### Worker-Done File (No Hook — Active Polling Required)

Worker exit creates `/tmp/worker-<name>.done` but **no PostToolUse hook is configured to detect it**. The file exists for any future hook integration but is currently unread. Active polling via `worker-cli status` is required — Opus is NOT notified automatically when a worker exits or hits the context limit.

### Worker Death Recovery — Worker-to-Worker Handoff (Always)

When a worker dies mid-task or mid-recap: ALWAYS spawn a successor. ALWAYS handoff. The dying worker's commits + SUCCESSOR-HANDOFF note in the last commit message body contain everything the successor needs. Opus does NOT do file archaeology.

The dying worker's SUCCESSOR-HANDOFF format is defined in `~/.claude/shared-rules/worker/worker-rules.md` § 6 — Opus does not duplicate it.

**Opus's role on detected worker death:**
1. Verify the dying worker has at least one commit on its branch: `git -C <worktree> log --oneline -3`
2. Spawn fresh successor: `worker-cli spawn <successor-name> /tmp/prompt.md <project_path> sonnet`
3. Successor's prompt is short: "You are a successor worker. Read the latest commit message on branch `<dying-worker-branch>` — it contains a SUCCESSOR-HANDOFF block. Resume from the exact point described. First action: print the handoff content back as confirmation before doing any work."
4. Phase 2 Cross-Model check on the successor's first response — does it match the dying worker's handoff intent?

**Pre-spawn safety**: if the dying worker has ZERO commits (died during read-phase before any work landed), the successor's prompt becomes the ORIGINAL task prompt — not a handoff resume. Same as a normal initial spawn. This is the only edge case; everything else is handoff.

Only kill the dying worker (and remove its worktree) AFTER the successor has confirmed handoff understanding and started its first commit. `worker-cli kill` is irreversible.


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

**Scope (concern separation):** Opus session-end RECAP covers ONLY:
1. **Files Opus touched directly** — rule files in `~/.claude/`, beads, RAG sync, cross-project edits (Worker Project Scope rule: workers only touch the current project, anything cross-project is Opus's)
2. **Worker omissions Opus noticed** — drift Opus spotted during Phase 4 Review or post-merge verification that the worker missed. Document the omission as a session-end fix.

That's it. Workers do worker recaps for their tasks (per `~/.claude/shared-rules/worker/worker-rules.md` § 6 — fully self-contained). Opus does Opus recap for Opus's surface. **Opus NEVER recaps a worker's task surface** — if a worker recap was incomplete, the fix is to either (a) catch it in Phase 4 Review and dispatch a follow-up worker, or (b) note the omission in Opus session-end recap WITHOUT redoing the worker's job.

Opus NEVER deliberately moves drift to session-end. If a session-end RECAP finds substantial drift from a completed worker task that the worker should have covered, that's a process violation — investigate why Phase 4 Review didn't catch it and adjust next session.

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
