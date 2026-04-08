# Workers (continued 2)

## Phase 5: Merge + Lifecycle

### Merging

`worker_merge(name)` merges the branch into current branch (`dev`). Worker stays alive.

**Post-Merge Verification (MANDATORY):**
- If merge says "Already up to date" → STOP. Worker did NOT commit. Investigate via `worker_capture`.
- Run `git diff HEAD~1 --name-only` — check expected files are modified
- If no changes: `worker_send` with commit instructions

**After merge:** Verify — run tests, MCP tool calls, screenshots, check integration.

**"Verified" ≠ Actually Tested:**
- Worker Completion Checklists saying "verified" are claims, not proof
- Workers in worktrees may lack venv, MCP tools — their "tests" may never have run
- After merging: run actual test commands yourself

Concrete failure (2026-03-16): 3 workers claimed "verified" but had no venv in worktree — tests never ran.

### Worker Stays Alive

**Workers stay alive until their feature is verified by the user.**

1. Worker implements → commits → outputs Completion Checklist → goes idle
2. Opus reviews (Phase 4) → merge
3. Opus/User verifies feature live (start app, run tests, screenshot)
4. Bug found → `worker_send` with bug info → worker fixes with full context
5. Verification passes → user approval → THEN kill in RECAP

**No worker kill before RECAP.** The cost of keeping a tmux session alive is zero. The cost of re-spawning with lost context is enormous.

**Cross-session workers:** Document alive workers in Bead STAND block: worker name, what it did, what to verify. Next session uses `worker_list` + `worker_capture` to interact.

### Killing Workers

**NEVER kill without checking `worker_status` first.**
- `working` → do NOT kill
- `idle` → safe to kill (or send follow-up)
- `exited` → tmux session can be cleaned up

**Status Detection:** Uses `#{window_activity}` timestamp. Threshold: 10 seconds without output → idle.

### Worker-Done Notification (Automatic)

Worker exits → `/tmp/worker-<name>.done` created → PostToolUse hook detects it → system hint to Claude. No manual notification needed.

---

## Quick Reference: The 5 Phases

```
Phase 1: DISPATCH    — Abstract task + "how would you solve this?"
Phase 2: EVALUATE    — Critically assess worker's proposed approach
Phase 3: GO          — Worker implements, correct via worker_send if needed
Phase 4: REVIEW      — Read changed files, verify code quality
Phase 5: MERGE       — Merge, verify live, worker stays alive until RECAP
```

**The key insight:** Opus has codebase understanding from PLAN. Workers have implementation focus. Opus evaluates worker proposals against its mental model — not blindly accepting, not micromanaging with exact code. The worker figures out HOW, Opus judges WHETHER it's right.
