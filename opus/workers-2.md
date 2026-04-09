# Workers (continued)

## Phase 2: Bewerten — Worker-Vorschlag evaluieren

After dispatching, the worker reads files and proposes an approach. Read via `worker_capture(tail=30-50)`.

**Evaluate critically — Opus has the codebase context:**
- Does the worker's approach address the RIGHT problem?
- Does it match existing patterns you saw in PLAN phase?
- Is the approach simpler or more complex than necessary?
- Does it miss files or edge cases you know about?

**Decision:**
- Worker's approach is correct → `worker_send` with "Go, implement it."
- Worker's approach is wrong → `worker_send` with correction: "Your approach misses X. Instead, focus on Y because Z."
- Worker challenges your expectation with a BETTER idea → evaluate honestly, give Go if convinced.

**Prohibited:**
- Accepting worker proposals at face value without critical evaluation
- "Looks good" without checking against your mental model
- Merging after reading only the Completion Checklist without reviewing actual code

Concrete failure (2026-04-05): hooks-redesign Worker implemented noise-filter and persisted-file-loading — both valid features, but neither addressed the core problem. Opus had no mental model and couldn't recognize the misalignment.

---

## Phase 3: Go + Implementation

Worker implements after receiving Go. During implementation:

### While Workers Run

**Do NOT poll status repeatedly.** Workers take 2-10 minutes.

**Pattern:**
1. Spawn worker → do independent work (rule edits, other planning, exploration)
2. Background timer fires → `worker_status` → if idle, proceed with Phase 4
3. Use `worker_capture(tail=N)` to read output after worker goes idle

### worker_send Scope Compliance (NON-NEGOTIABLE)

**Worker prompts MUST match what was agreed with the user.** If user says "teste nur Stealth" → worker_send contains ONLY Stealth changes. No additional settle_seconds, no Tor, no extras.

Opus tendency: "while we're at it, also test X" → adds variables the user didn't ask for → results uninterpretable → user corrects.

**Before every worker_send:** Re-read what was agreed. Does the prompt match? If not, cut.

Concrete failure (2026-04-07): User said "wir testen nur stealth". Opus sent worker 7-test matrix including settle_seconds + Tor + stealth patches. Worker changed variables the user explicitly excluded. User corrected 2x.

### Reusing Workers (worker_send) — AGGRESSIVE REUSE

Workers retain full context. After completion, send follow-up tasks via `worker_send` instead of spawning new workers.

**ALWAYS prefer `worker_send` over new spawn.** The threshold for spawning a new worker is HIGH:
- Idle worker exists that touched the same files/domain → `worker_send`. No exceptions.
- Follow-up task builds on previous work → `worker_send`. Even if the task feels "different" (e.g., switching from pydoll to patchright testing in the same test suite).
- New spawn is ONLY justified when: (a) no idle worker has relevant context, OR (b) idle worker is below 30% context.

**Before EVERY `worker_spawn`:** Check `worker_list`. If ANY idle worker has context overlap with the new task → use `worker_send`. Ask yourself: "Does an existing worker already know these files?" If yes → reuse.

**Context Budget Rule:** Below 30% context remaining → do NOT send follow-ups. Worker can die mid-task. Above 30%: follow-ups OK for small tasks. For large follow-ups, spawn fresh worker.

Concrete failure (2026-04-09): engine-tuning worker idle at 41% context, knows entire test suite (engine_selectors.py, stealth_config.py, 27/28 scripts). Opus spawned new patchright-test worker for a Brave test that uses the same directory and queries. User corrected: "warum ein neuer worker?" — engine-tuning should have received the task via worker_send.

### Course Correction

If worker delivers something misaligned:
1. `worker_capture` → read what they did
2. Identify the gap between expected and actual
3. `worker_send` with specific correction: "You did X but the requirement is Y. Please change Z."
4. NOT: spawn a new worker (wastes context, loses the worker's understanding)

Concrete failure (2026-04-04): Spawned 6 workers for hooks-pane UI. Should have been 1-2 workers with `worker_send` for corrections. Each new worker lost the previous worker's context.

---

## Phase 4: Review

After worker goes idle, review BEFORE merging.

### Code Review (MANDATORY)

1. `worker_capture(tail=30-50)` → read Completion Checklist
2. Read EVERY changed file in the worktree: `Read /path/to/.claude/worktrees/<name>/src/<file>`
3. Check: correctness, existing patterns followed, no regressions
4. If issues found → `worker_send` with fix instructions, back to Phase 3
5. If review passes → proceed to Phase 5

**What to check:**
- Does the code address the actual problem (not a symptom)?
- Does it follow existing patterns in the codebase?
- Are there uncommitted local changes that conflict?
- Did the worker commit? (Check Completion Checklist)

**Glue work before merge:** Copy gitignored files, extract configs — anything that lives only in the worktree. Merge deletes the worktree, anything not saved is lost.

Concrete failure (2026-03-25): engagement-agent Worker rewrote `.claude/agents/engagement-finder.md` (gitignored). Worktree deleted on merge. File had to be rescued via /tmp copy.
