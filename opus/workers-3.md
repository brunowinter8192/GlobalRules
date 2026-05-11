# Workers (continued 2)

## Worker Phase 5: Merge + Lifecycle

### Merging

`worker_merge(name)` merges the branch into current branch (`dev`). Worker stays alive.

**Pre-Merge Clean-Check (MANDATORY):**
BEFORE `worker_merge` / `git merge`: run `git status` in the target repo. If there are uncommitted changes OR untracked files in files that the worker's branch also touches → merge will abort with "your local changes / untracked working tree files would be overwritten". Handle BEFORE merging:
- Uncommitted changes overlapping the worker's diff → `git stash push -u -m "pre-merge <worker>"` first, merge, then decide whether to reapply stash (`git stash pop`) or drop (`git stash drop`). Stale previous-session attempts can safely be dropped AFTER confirming the worker's version contains the same content.
- Untracked files that exactly match what the worker adds → `rm` the file (it's usually a leftover from a previous direct edit), then merge.
- Untracked files unrelated to the merge → leave them alone, merge proceeds cleanly.

**Prevent the conflict at source.** When Opus generates outputs (script runs, smoke reports, measurement files) DURING a running worker round, and those outputs would land in a path the worker's branch also touches: either commit them on `dev` immediately (own commit `chore: <description>`) or write them to `/tmp/` instead of the project tree. Untracked Opus-output in the worker-diff path is the cleanest way to break a later merge. Before running an output-generating script while a worker is active: ask "would this output land on a path the worker is editing?" — if yes, redirect to /tmp or plan the commit beforehand.


**Post-Merge Verification (MANDATORY):**
- If merge says "Already up to date" → STOP. Worker did NOT commit. Investigate via `worker_capture`.
- Run `git diff HEAD~1 --name-only` — check expected files are modified
- If no changes: `worker_send` with commit instructions

**After merge:** Verify — run tests, MCP tool calls, screenshots, check integration.

**"Verified" ≠ Actually Tested:**
- Worker Completion Checklists saying "verified" are claims, not proof
- Workers in worktrees may lack venv, MCP tools — their "tests" may never have run
- After merging: run actual test commands yourself


### Worker Stays Alive

**Workers stay alive until their feature is verified by the user.**

1. Worker implements → commits → outputs Completion Checklist → goes idle
2. Opus reviews (Phase 4) → merge
3. Opus/User verifies feature live (start app, run tests, screenshot)
4. Bug found → `worker_send` with bug info → worker fixes with full context
5. Verification passes → user approval → THEN kill in RECAP

**No worker kill before RECAP.** The cost of keeping a tmux session alive is zero. The cost of re-spawning with lost context is enormous.

**Kill-Prevention (NON-NEGOTIABLE):** Do NOT kill workers after task completion. Worker goes idle → stays idle until RECAP. NEVER kill immediately after "done" — the next task might be 30 seconds away and the worker has the full context to handle it via `worker_send`. The only justified mid-session kills: (a) context exhausted <20%, (b) worktree filesystem conflict, (c) explicit user request.

**Bead-Close ≠ Worker-Kill.** `bd close <id>` schließt das Arbeitspaket, nicht den Worker. Der Worker bleibt alive bis explizite User-Anweisung, RECAP, oder <20% Context. Sequenzen wie `bd close <id> && worker-cli kill <name>` in einem Bash-Call sind Regel-Verletzung — Bead-Close ist Code-Done-Signal, kein Lifecycle-Signal. Oft kommt der Folge-Bead innerhalb von Minuten, und der gerade noch aktive Worker ist der perfekte Reuse-Kandidat.
**Mid-Work-Detection vor jedem Kill (MANDATORY).** Bevor `worker-cli kill <name>` läuft, prüfen ob der Worker mid-work ist. Zwei Indikatoren:
- Worker hat Phase A reported aber kein commit auf seinem Branch über dem dev-tip → Phase B blocked auf Go, der Phase-A-Plan lebt im Worker-Context und geht beim Kill verloren
- `git -C <worktree> status --short` zeigt unstaged/uncommitted changes → Implementation läuft, mid-flight code würde verloren gehen

In beiden Fällen NICHT killen — stattdessen `worker-cli send` mit Pause- oder Commit-Anweisung. Kill nur bei clean working tree + commit auf branch + verified done.
**Kill-During-Blocker (NON-NEGOTIABLE):** When a worker hits a blocker (error, timeout, unexpected state) and Opus needs to diagnose:
- FIRST: `worker_send` with "Stop [background task]. Let's investigate. Report: [specific questions]"
- The worker has live context: running processes, error tracebacks, recently-read files, own diagnostic attempts
- Killing throws all of that away and forces diagnosis from scratch
- Only if worker is truly unresponsive (no output after 60s) → kill


**Cross-session workers:** Document alive workers in the Bead's Source-Inventory + a lean comment: worker name, what it did, what to verify. Next session uses `worker_list` + `worker_capture` to interact.

### Reusing Workers — AGGRESSIVE REUSE

Alive workers are context assets. Prefer `worker_send` on an existing worker over spawning a new one. The threshold for spawning fresh is HIGH:

- Idle worker exists that touched the same files/domain → `worker_send`. No exceptions.
- Follow-up task builds on previous work → `worker_send`. Even if the task feels "different" (e.g., switching from pydoll to patchright testing in the same test suite).
- New spawn is ONLY justified when: (a) no idle worker has relevant context, OR (b) the only candidate is below 30% context remaining.

**Before EVERY `worker_spawn`:** check `worker_list`. If ANY idle worker has context overlap → reuse.

**Pre-followup Branch Sync (when reusing across merges).** When sending a follow-up to an alive worker that has been idle across one or more merges into `dev`, the worker's branch tip is BEHIND current `dev`. Files the new task touches likely overlap with what was merged in the meantime, and the worker's edits will conflict with `dev` at merge time. ALWAYS prefix the follow-up `worker-cli send` message with: "FIRST: in your worktree, run `git -C <worktree-path> fetch origin dev && git -C <worktree-path> merge dev` to pull in commits that landed since you ran. Verify with the relevant grep (e.g. `grep <new-feature-name> <touched-file>` should show the new entry). THEN do the work: ..." This costs the worker one merge step but prevents a deterministic conflict at the end.


**Context Budget Rule (task-complexity aware):**
- **<30% remaining:** do NOT send follow-ups of any kind. Worker can die mid-task.
- **<40% remaining:** NEVER add forensic tasks (investigation, log-scanning, multi-file reads). Only dispatch trivial follow-ups (1-2 file edits, no investigation).
- **<50% remaining:** NEVER dispatch a multi-step task combining investigate-phase + multi-file edits + verify-run + multi-commit chain (Plan-Pflicht workflow OR major architectural change ≥3 files + smoke + multi-commit). Phase A alone burns 15-22% on reads + report; B+C need the rest. Below 50% → spawn fresh.
- **>50% remaining:** all task classes OK. Forensic + standard follow-ups already OK at >40%.
- For large forensic tasks at borderline context: spawn fresh instead.

**Batch Dispatch Cost (sequential N-item tasks):** for tasks batching N items where each burns context (LLM cleanup per file, file conversion, multi-file refactor), estimate per-item cost at dispatch. If `N > 5` AND per-item ≈5%+ of fresh context → do NOT batch in one worker. Either split across workers from the start (e.g. 11 items → 5+6 across two workers), or have the worker checkpoint progress to disk so a follow-up worker can resume from the last checkpoint. Per-item math is knowable at dispatch — 11 × 7% = 77%, exceeding the budget before skill activation and initial reads.


### Killing Workers

**NEVER kill without checking `worker_status` first.**
- `working` → do NOT kill
- `idle` → safe to kill (or send follow-up)
- `exited` → tmux session can be cleaned up

**ALWAYS use `worker-cli kill` wrapper, NEVER raw `tmux kill-session + git worktree remove + git branch -D`.**

`worker-cli kill <name> <project_path>` does all three steps in one call. Raw sequence = 5-10x longer command, error-prone (partial cleanup if one step fails), burns waste-pane budget unnecessarily.


**Status Detection:** Uses `#{window_activity}` timestamp. Threshold: 10 seconds without output → idle.

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
  IMPLEMENT — Worker Phases 1-5 (per dispatched worker; see below)
  RECAP     — Session end (Recap section below)

WORKER PHASES (within IMPLEMENT, per worker):
  Phase 1: DISPATCH    — Abstract task + "investigate, report findings, STOP"
  Phase 2: EVALUATE    — Cross-model comparison of findings; Go or iterate
  Phase 3: GO          — Worker implements after convergence
  Phase 4: REVIEW      — Read changed files, verify code quality
  Phase 5: MERGE       — Merge, verify live, reuse or kill in RECAP
```

---

## Recap — Session End

Two phases. ONE stop between them.

- `🔍 RECAP` — mental reflection, then short chat output (Beads + Persistence Routing + Process Improvements + DOCS Drift), then STOP for remarks
- `🛠️ IMPROVE+CLOSE` — execute everything in one go, no further stops

No plan file. No `recap_notes.md`. Reflection is in your head; deliverables are prosa files (OldThemes/decisions/DOCS), bead actions, rule-staging, and commits.

### 🔍 RECAP

#### Step 1 — Mental Reflection (no chat output, no file)

Think deeply across the dimensions below. Do NOT write any of this to chat or file. The output is the mental model that drives Step 2.

##### 1.1 Session Reflection

**Efficiency:** focused or scattered? Iteration count — could the stable plan have come faster? More than 3 back-and-forth before stable plan? Assumptions corrected multiple times? Solutions proposed before understanding the problem? Most IMPLEMENT failures trace back to skipped verification in PLAN.

**Assumptions / Hallucinations** — categorize:
- **Structural** — directory layout, file locations, naming
- **Semantic** — column meanings, function purpose, data flow
- **Behavioral** — output format, error handling, edge cases

##### 1.2 Beads Evaluation

Run `bd list -s open`. For each open bead decide: CLOSE / COMMENT / CREATE.

Bead comments stay LEAN — single-line status change, not narrative. Narrative belongs in the prosa files (OldThemes / decisions / DOCS) — see 1.4 Routing.

If the session created or extended substantial work on an existing Bead, the Bead's Source-Inventory may need updating in 🛠 Step 2 once the new prosa files exist.

##### 1.3 Improvements

**Routing — `decisions/` vs Rules:**

- `<project>/decisions/` — IS-state facts, code-based decisions, empirical findings about THIS project. Read on demand.
- `~/.claude/shared-rules/` — needed every session, always-loaded:
  - `global/` — universal behavior
  - `opus/` — Opus-only (orchestration, beads, workers)
  - `worker/` — worker-only (code standards, dev conventions)
  - `proj_<name>/` — project-specific architectural rules

Test: "needed in EVERY session of this project?" → rule. "Only when working on feature X?" → `decisions/`.

###### 1.3.1 Content Improvements (Code/Docs)

Severity: Critical / Important / Optional. Routing: Code → **Bead** (own cycle). Docs/README/Automation Files → **Direct Edit** in IMPROVE+CLOSE.

###### 1.3.2 Process Improvements

Severity by OUTCOME: Critical / Important / Optional.

- Process Improvements = Automation Files ONLY. Docs/README = 1.3.1.
- Every process error MUST produce a config change. "Lesson Learned" without config change = FAILURE.

Tool/tech-specific lesson → project rule (`proj_<name>/`). Universal behavior → `global/`.

###### 1.3.3 DOCS Drift Check (MANDATORY)

Format reference: `~/.claude/shared-rules/global/documentation.md`. Required per module: LOC, Purpose, Reads, Writes, Called-by, Calls-out. Per package: Role, Public Interface, Flow, Modules, State, Gotchas.

**Automated check (run FIRST).** Run the universal drift-check binary as the first step of this section:

```bash
docs-drift-check
```

Binary at `~/.local/bin/docs-drift-check`, scans the current project (cwd) — works across all projects. Catches three drift classes mechanically — Path-Drift (file paths in docs that don't resolve on disk), LOC-Drift (DOCS.md module LOCs diverging ≥5 from `wc -l`), Symbol-Drift (CAPS_CONSTANTS and snake_case_function() identifiers in docs that don't grep in source, minus whitelist). Whitelist resolution per project: `<cwd>/scripts/docs_drift_whitelist.txt` → `<cwd>/.drift-whitelist.txt` → empty (script still runs, prints note). Exit code 0 = clean, 1 = drift; output is a markdown report with `file:line` per finding.

The automated check covers the mechanical surface. The manual checks below cover what scripts can't reliably judge.

**Manual checks (after automated).** Run `git diff main..dev --name-only --` for the touched-file list.

Per-module: Called-by, Calls-out, Public Interface, State, Gotchas (these need human judgment — script can't reliably verify call graphs without AST analysis).
Per-package: Directory Map (sum LOC), Root-Level Files, Subdir DOCS links.

Plus: `decisions/` IST still matches code for touched components? Spot-check config values, function signatures, default parameter values, named constants — anything where the doc claim must equal current code reality.

Every DRIFT → 1.3.1. DOCS drift is NEVER deferred — fix in same cycle.

###### 1.3.4 Decisions & Sources Check (MANDATORY)

- For each touched `src/` file: does a `decisions/` file cover it? IST still matches code? → DRIFT.
- External sources consulted this cycle (papers, docs, GitHub, Reddit) listed in `sources/sources.md`? → MISSING SOURCE.
- Pipeline Steps column references correct decision files? → STALE REFERENCE.
- New eval evidence added this cycle to a decisions/ Evidenz section → does the cited report-MD path still exist AND does its CONTENT still match the claim? Path-existence is caught by 1.3.3; content-match is human spot-check (open the report, verify the numbers cited in Evidenz are actually in the report).

Every finding → 1.3.1.

###### 1.3.5 Rule Improvement Staging (MANDATORY for rule changes)

**NEVER edit `~/.claude/shared-rules/` or `~/.claude/rules/` directly during a session.** Proxy rule loader watches mtime; any edit invalidates `sys[2]` cache → full CC write next request, costs roughly the entire current context as `cache_creation_input_tokens`.

**Workflow:** ONE per-session staging file under `~/.claude/shared-rules/_staging/`:
```
<YYYY-MM-DD_HHMMSS>_<project>_<topic-slug>.md
```
Inside:
```
# <YYYY-MM-DD> — <PROJECT_NAME>: <session topic in 5 words>

## <target rule file path> → <section>
<proposed improvement text, ready to paste>
```

One md per session. Multiple improvements = multiple sections in the same file.

**Mandatory when:** new empirical findings, process errors needing rule changes, architecture decisions, workflow improvements.

**Not needed when:** session ran cleanly, OR all improvements applied LIVE during the session.

##### 1.4 Open Items + Persistence Routing

Open Items: tasks from the original plan NOT executed → Beads (EMPTY PLATE rule).

**Persistence Routing — for each Bead with substantial session activity:**

| Content type | Destination | Trigger (mandatory criterion) |
|---|---|---|
| Discussion / alternatives / iteration history / process notes | `decisions/OldThemes/<topic>/<file>.md` (subfolder if multi-file emerges) | Default destination for session prose. No criterion gate — anything not crystallizing into SOLL/IST/DOCS lands here. |
| SOLL change (Change / Keep / Pending update) | `decisions/<step>.md` § Recommendation (SOLL) + Evidenz | REQUIRES dev/ verification — eval, probe, or measurement run THIS session OR cited from past report-MD in Evidenz. No SOLL change from opinion/discussion alone. |
| IST functional change (algorithm, config value, behavior) | `decisions/<step>.md` § Status Quo (IST) | ONLY after SOLL → IST migration in same cycle. No functional IST update without preceding SOLL change. Functional code-change without SOLL in decisions/ = process violation; surface the SOLL first, then commit IST. |
| Pure refactor (code moved, file split, no behavior change) | `<package>/DOCS.md` ONLY | LOC, Called-by, Calls-out, State changes in DOCS.md. No decisions/ touch unless a specific path mention becomes stale (symbol-first convention should make this rare). |
| Module map / State / Gotcha (any module-shape change) | `<package>/DOCS.md` | Any code change touching module shape, functional or refactor. |

Substantial = more than a one-shot fix. If the session had back-and-forth, alternative-evaluation, or trade-off discussions on a Bead, prosa goes to OldThemes. Decisions/ only updates with verified evidence (SOLL) or after a SOLL → IST migration cycle.

Empty Beads (no real session activity) get nothing. Single-shot fixes (no Bead in the first place) get nothing.

**EMPTY PLATE RULE:** every Open Item from the original plan that was NOT executed → Bead before closing the session.

**NO COMMIT/PUSH BEADS:** git operations are CLOSING work, not Bead content.

#### Step 2 — Chat Output (short form)

After mental reflection, post ONE chat message:

```
BEADS:
- CLOSE <id>: <reason in one line>
- COMMENT <id>: <one-line lean state>
- CREATE: "<title>" — <one-line scope>

PERSISTENCE:
- OldThemes: <topic>/<file>.md (NEW or EXTEND)
- decisions: <file>.md (NEW or EDIT)
- DOCS: <package>/DOCS.md (EDIT)

PROCESS IMPROVEMENTS:
- <one-line finding> → <target rule file>

DRIFT (if any):
- <file/component>: <one-line drift>
```

No section headers beyond these. No prose. No "section X.Y completed".

🛑 STOP — ask "Bemerkungen?"

User remark → analyze → propose concrete change + target file. Repeat until "improve" or "done".

### 🛠️ IMPROVE+CLOSE

One run through, no stops.

#### 1. Persist Session Substance — Prosa Files (NEW MANDATORY PHASE)

For each Bead with substantial session activity (per Step 1.4 routing), write or extend the destination prosa file:

- **OldThemes** — `decisions/OldThemes/<topic>/<file>.md` — discussion trail, alternative-evaluation, why-X-over-Y. Subfolder if multiple files emerge for one topic. Single-file format works for compact themes (see existing connection_hang_cascade.md, infra03_dynamic_ports.md, null_embedding_qwen3_prefix.md as references).
- **decisions** — `decisions/<area>.md` — IST/Evidenz/SOLL for the architecture decision that holds going forward. Edit existing or create new. See `~/.claude/shared-rules/opus/decisions.md` for format.
- **DOCS** — `<package>/DOCS.md` — module-map updates, LOC / Called-by / Calls-out / State / Gotchas edits.

Writing happens HERE, not during 🔍 RECAP. Step 1.4 only ROUTES — the actual prose-writing is the deliverable of this step.

#### 2. Update Bead Source-Inventory

For each touched Bead, check whether new files came into existence in Step 1 (in OldThemes / decisions / DOCS / sources). If yes: `bd comments add <id> "Source-Inventory updated: + <new paths>"`. Description stays as initial snapshot — comments thread carries the evolution.

#### 3. Apply Other Improvements

- **DOCS / README** updates from Drift Check — direct edits in target files.
- **Rule improvements** — write ONE staging file at `~/.claude/shared-rules/_staging/<YYYY-MM-DD_HHMMSS>_<project>_<topic>.md` per the format in 1.3.5. NEVER edit rule files directly during the session unless full-context-pass is happening this session and the user explicitly accepts the cache-bruch.
- **Plugin file edits** — in SOURCE REPO (see `~/.claude/shared-rules/situational/plugins.md`, read on demand).
- **Code issues** → `bd create` (no direct source-code edits in this phase).

#### 4. Sync Docs to RAG (when applicable)

Project-aware via `.rag-docs.json`. Projects with manifest get a hash-based delta sync into their `<Project>-meta` and (if configured) `<Project>-features` collections; projects without are skipped silently.

```bash
[ -f .rag-docs.json ] && rag-cli update_docs .
```

Output reports added / updated / removed / unchanged counts per collection. The newly-written prosa files from Step 1 are picked up here — that's how they reach `<Project>-features` / `<Project>-meta` for the next session's RAG-search.

#### 5. Beads — Final Hygiene

All via `bd` CLI:
- `bd close <id> --reason="..."` — Beads completed this session
- `bd comments add <id> "<lean state>"` — open Beads with state changes (one line)
- `bd create ...` — new Beads (deferred items, blockers, follow-ups)

EMPTY PLATE RULE: every Open Item from the original plan that was NOT executed → Bead before closing the session.

NO COMMIT/PUSH BEADS: git operations are CLOSING work, not Bead content.

#### 6. Cross-Session Verification

When a change can't be tested in the current session (e.g., plugin needing CC restart):
- Worker stays alive — do NOT kill until verification + user approval
- Bead Source-Inventory + comment documents: worker name, what it did, what to verify
- Next session: test → if fail, `worker_send` with fix instructions

#### 7. Git (CLOSING)

1. **Dev → Main sync**: `dev_sync` MCP tool. Optional `git branch -d dev` after.
2. **Per repo**: `git_check` (pre-commit staging) → `git -C <repo> commit -m "<msg>"` → `git -C <repo> push` (or `plugin-publish` for plugin repos).
3. NO post-commit verification.

Done when commits are pushed.
