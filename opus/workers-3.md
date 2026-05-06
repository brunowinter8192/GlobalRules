# Workers (continued 2)

## Phase 5: Merge + Lifecycle

### Merging

`worker_merge(name)` merges the branch into current branch (`dev`). Worker stays alive.

**Pre-Merge Clean-Check (MANDATORY):**
BEFORE `worker_merge` / `git merge`: run `git status` in the target repo. If there are uncommitted changes OR untracked files in files that the worker's branch also touches → merge will abort with "your local changes / untracked working tree files would be overwritten". Handle BEFORE merging:
- Uncommitted changes overlapping the worker's diff → `git stash push -u -m "pre-merge <worker>"` first, merge, then decide whether to reapply stash (`git stash pop`) or drop (`git stash drop`). Stale previous-session attempts can safely be dropped AFTER confirming the worker's version contains the same content.
- Untracked files that exactly match what the worker adds → `rm` the file (it's usually a leftover from a previous direct edit), then merge.
- Untracked files unrelated to the merge → leave them alone, merge proceeds cleanly.

**Prevent the conflict at source.** When Opus generates outputs (script runs, smoke reports, measurement files) DURING a running worker round, and those outputs would land in a path the worker's branch also touches: either commit them on `dev` immediately (own commit `chore: <description>`) or write them to `/tmp/` instead of the project tree. Untracked Opus-output in the worker-diff path is the cleanest way to break a later merge. Before running an output-generating script while a worker is active: ask "would this output land on a path the worker is editing?" — if yes, redirect to /tmp or plan the commit beforehand.

Concrete failure (2026-04-21, searxng): measurement-smoke fired by Opus while worker on refactor branch ran Round 2. Report `smoke_20260421_193051.md` landed as untracked in main tree. Subsequent merge: "The following untracked working tree files would be overwritten by merge". Manual cleanup + re-merge required.

Concrete failure (2026-04-14 evening): `rag-cli-convert` worker's merge into `MCP/RAG/dev` aborted twice. First abort: untracked `cli.py` in main (timestamp matched an earlier direct-edit attempt by the user pre-session). Second abort: modified `plugin.json`, `agents/rag-search.md`, `requirements.txt`, `skills/agent-rag-search/SKILL.md` — a complete stale MCP→CLI conversion attempt sitting uncommitted in main. Opus recovered via `rm cli.py && git stash push -u && git merge`, but a pre-merge `git status` check would have caught it in one pass and made the cleanup intent explicit.

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

**Kill-Prevention (NON-NEGOTIABLE):** Do NOT kill workers after task completion. Worker goes idle → stays idle until RECAP. NEVER kill immediately after "done" — the next task might be 30 seconds away and the worker has the full context to handle it via `worker_send`. The only justified mid-session kills: (a) context exhausted <20%, (b) worktree filesystem conflict, (c) explicit user request.

**Bead-Close ≠ Worker-Kill.** `bd close <id>` schließt das Arbeitspaket, nicht den Worker. Der Worker bleibt alive bis explizite User-Anweisung, RECAP, oder <20% Context. Sequenzen wie `bd close <id> && worker-cli kill <name>` in einem Bash-Call sind Regel-Verletzung — Bead-Close ist Code-Done-Signal, kein Lifecycle-Signal. Oft kommt der Folge-Bead innerhalb von Minuten, und der gerade noch aktive Worker ist der perfekte Reuse-Kandidat. Concrete failure (2026-04-27): meta-legend Worker direkt nach Bead-Close gekillt im selben Bash-Call. User: "warum hast du denn den worker einfach gekillt".

**Mid-Work-Detection vor jedem Kill (MANDATORY).** Bevor `worker-cli kill <name>` läuft, prüfen ob der Worker mid-work ist. Zwei Indikatoren:
- Worker hat Phase A reported aber kein commit auf seinem Branch über dem dev-tip → Phase B blocked auf Go, der Phase-A-Plan lebt im Worker-Context und geht beim Kill verloren
- `git -C <worktree> status --short` zeigt unstaged/uncommitted changes → Implementation läuft, mid-flight code würde verloren gehen

In beiden Fällen NICHT killen — stattdessen `worker-cli send` mit Pause- oder Commit-Anweisung. Kill nur bei clean working tree + commit auf branch + verified done. Concrete failure (2026-04-29, Monitor_CC RAM): copy-button v1 Worker hatte Phase A komplett (column-routing + ⎘-Symbol + ✓-flash design), Phase B war auf Go geblockt wegen file-overlap. Während eines Branch-Divergenz-Fixes hat Opus vier idle Worker auf einmal gekillt inklusive copy-button — der ganze Phase-A-Plan war damit weg. v2-Spawn mit baked-in Spec aus Original-Phase-A-Report nötig, doppelte Spawn-Cost.

**Kill-During-Blocker (NON-NEGOTIABLE):** When a worker hits a blocker (error, timeout, unexpected state) and Opus needs to diagnose:
- FIRST: `worker_send` with "Stop [background task]. Let's investigate. Report: [specific questions]"
- The worker has live context: running processes, error tracebacks, recently-read files, own diagnostic attempts
- Killing throws all of that away and forces diagnosis from scratch
- Only if worker is truly unresponsive (no output after 60s) → kill

Concrete failure (2026-04-16, searxng): `tier-eval` worker killed mid-CREATE_TARGET-investigation instead of a `worker_send` to stop the run and investigate together. Re-investigation required fresh worker, fresh file reads, re-hypothesizing. User: "warum spawnst du die scheiß komplette zeit neue worker obwohl wir immer warten wollen bis ein worker keinen context mehr hat".

**Cross-session workers:** Document alive workers in Bead STAND block: worker name, what it did, what to verify. Next session uses `worker_list` + `worker_capture` to interact.

### Reusing Workers — AGGRESSIVE REUSE

Alive workers are context assets. Prefer `worker_send` on an existing worker over spawning a new one. The threshold for spawning fresh is HIGH:

- Idle worker exists that touched the same files/domain → `worker_send`. No exceptions.
- Follow-up task builds on previous work → `worker_send`. Even if the task feels "different" (e.g., switching from pydoll to patchright testing in the same test suite).
- New spawn is ONLY justified when: (a) no idle worker has relevant context, OR (b) the only candidate is below 30% context remaining.

**Before EVERY `worker_spawn`:** check `worker_list`. If ANY idle worker has context overlap → reuse.

**Pre-followup Branch Sync (when reusing across merges).** When sending a follow-up to an alive worker that has been idle across one or more merges into `dev`, the worker's branch tip is BEHIND current `dev`. Files the new task touches likely overlap with what was merged in the meantime, and the worker's edits will conflict with `dev` at merge time. ALWAYS prefix the follow-up `worker-cli send` message with: "FIRST: in your worktree, run `git -C <worktree-path> fetch origin dev && git -C <worktree-path> merge dev` to pull in commits that landed since you ran. Verify with the relevant grep (e.g. `grep <new-feature-name> <touched-file>` should show the new entry). THEN do the work: ..." This costs the worker one merge step but prevents a deterministic conflict at the end.

Concrete failure (2026-05-04, searxng): `lobsters` worker was sent the SE-add + HN-drop task while `dev` had moved forward with the `scholar-eval` merge. Without the fetch+merge prefix, the worker's branch would have conflicted on `src/search/search_web.py` (ENGINES dict adjacent additions), `dev/search_pipeline/05_search_smoke.py` (AVAILABLE_ENGINES), `decisions/search05_engine_expansion.md`, and `CLAUDE.md`. The instruction was added pre-emptively — no conflict at merge time.

**Context Budget Rule (task-complexity aware):**
- **<30% remaining:** do NOT send follow-ups of any kind. Worker can die mid-task.
- **<40% remaining:** NEVER add forensic tasks (investigation, log-scanning, multi-file reads). Only dispatch trivial follow-ups (1-2 file edits, no investigation).
- **<50% remaining:** NEVER dispatch a multi-step task combining investigate-phase + multi-file edits + verify-run + multi-commit chain (Plan-Pflicht workflow OR major architectural change ≥3 files + smoke + multi-commit). Phase A alone burns 15-22% on reads + report; B+C need the rest. Below 50% → spawn fresh.
- **>50% remaining:** all task classes OK. Forensic + standard follow-ups already OK at >40%.
- For large forensic tasks at borderline context: spawn fresh instead.

**Batch Dispatch Cost (sequential N-item tasks):** for tasks batching N items where each burns context (LLM cleanup per file, file conversion, multi-file refactor), estimate per-item cost at dispatch. If `N > 5` AND per-item ≈5%+ of fresh context → do NOT batch in one worker. Either split across workers from the start (e.g. 11 items → 5+6 across two workers), or have the worker checkpoint progress to disk so a follow-up worker can resume from the last checkpoint. Per-item math is knowable at dispatch — 11 × 7% = 77%, exceeding the budget before skill activation and initial reads.

Concrete failure (2026-05-05, searxng arch dispatch): hc3-html-unescape worker reused at 35% context for v2+v3 redesign (4 production files + 2 docs + smoke run + 4-commit chain). Worker died mid-implementation at ~25% with 220 lines of search_web.py uncommitted in worktree. Recovery: patch capture + fresh dl9-redesign spawn + re-do Phase A diff plan. Cost: ~30 min wall-clock + duplicate Phase A reads. <50% threshold from start would have spawned fresh.

Concrete failure (2026-04-30, RAG long-batch): pdfconv worker dispatched all 11 chunking-paper conversions in batch mode (MinerU + LLM cleanup + index-dir). Worker hit "Prompt is too long" between PDF 9 and 10, going idle without the final index step. Recovery required pdfconv2 for the remaining 2 PDFs + index. Per-PDF cost ≈ 7% × 11 = 77%, knowable at dispatch. 2-worker split (5+6) from the start would have completed cleanly.

Concrete failure (2026-04-09): engine-tuning worker idle at 41% context, knew the entire test suite (engine_selectors.py, stealth_config.py, 27/28 scripts). Opus spawned a new patchright-test worker for a Brave test in the same directory. Should have been a `worker_send` to engine-tuning.

Concrete failure (2026-04-09): fulltext-expand worker idle at 73% context after editing proxy_pane.py (tools expansion + system blocks). Next task — stripped-tool visualization + auto-load fix — touched the same file. Opus spawned strip-visual instead of reusing. Wasted a slot and lost all the proxy_pane.py context.

Concrete failure (2026-04-20): Spawned `gc-wrapper` worker in blank without running `worker_list` / `tmux ls | grep worker-blank` first. 4 stale `worker-blank-*` sessions from April 2-8 existed (likely dead, but check is the rule). User: "bitte worker reusen in zukunft". Rule violated even when no overlap was found — the CHECK is the enforcement point, not only the reuse decision.

Concrete failure (2026-04-19): warnings-zero worker at 38% context was sent dual-task (Pyright strip forensic + Glob pattern trivial). Task A required inspecting proxy_log JSONL for `<new-diagnostics>` pattern — forensic-depth task. Worker investigated wrong file (session JSONL attachment structure) for 5+ minutes, dropped to 18% context. Opus redirect came late. Worker finished only Task B, died mid-Task A with uncommitted code. Opus had to rescue the uncommitted work manually. Rule derived: at <40% remaining context, NEVER add forensic tasks — only trivial follow-ups. Investigation needed → fresh worker.

### Killing Workers

**NEVER kill without checking `worker_status` first.**
- `working` → do NOT kill
- `idle` → safe to kill (or send follow-up)
- `exited` → tmux session can be cleaned up

**ALWAYS use `worker-cli kill` wrapper, NEVER raw `tmux kill-session + git worktree remove + git branch -D`.**

`worker-cli kill <name> <project_path>` does all three steps in one call. Raw sequence = 5-10x longer command, error-prone (partial cleanup if one step fails), burns waste-pane budget unnecessarily.

Concrete failure (2026-04-20): Raw cleanup `tmux kill-session ... && tmux kill-session ... && git -C /long/path worktree remove ... && git -C /long/path worktree remove ... && git -C /long/path branch -D <2 branches>` = 576 chars for 2-worker cleanup. `worker-cli kill tabular-layout /path && worker-cli kill strip-fix /path` = ~130 chars, equivalent result.

**Status Detection:** Uses `#{window_activity}` timestamp. Threshold: 10 seconds without output → idle.

### Worker-Done Notification (Automatic)

Worker exits → `/tmp/worker-<name>.done` created → PostToolUse hook detects it → system hint to Claude. No manual notification needed.

### Worker Death Recovery

When a worker hits the context limit and dies mid-task, their worktree is still on disk with whatever uncommitted work they had. No work is lost unless Opus kills the worker before reviewing.

1. **Check the worktree for uncommitted work:** `git -C <worktree_path> status --short`
2. **Read and evaluate** uncommitted files — are they complete enough to be useful?
3. **Merge first** — run `worker_merge` for whatever the worker committed on their branch.
4. **Copy uncommitted recoverable files** from worktree into the target repo: `cp <worktree>/path/file <main>/path/file`
5. **Commit the copied files separately** with a message noting the recovery: `chore: recover from dead worker <name>`
6. **Drop debug artifacts** (temp scripts, debug prints, half-finished code) before committing.

Only kill the worker (and remove the worktree) AFTER this review. `worker_kill` is irreversible.

---

## Quick Reference: The 5 Phases

```
Phase 1: DISPATCH    — Abstract task + "investigate, report findings, STOP"
Phase 2: EVALUATE    — Cross-model comparison of findings; Go or iterate
Phase 3: GO          — Worker implements after convergence
Phase 4: REVIEW      — Read changed files, verify code quality
Phase 5: MERGE       — Merge, verify live, reuse or kill in RECAP
```
