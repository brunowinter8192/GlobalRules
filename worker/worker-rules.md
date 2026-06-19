
# Worker Rules — Worktree Isolation & Report

These rules apply to every worker session. Run the Pre-Edit Check to determine your mode.

## 1. Code Investigation — concrete files only

Investigate using the concrete files Opus names in your prompt (src/, decisions/, DOCS.md, dev/). Read those files directly. NEVER run `rag-cli` or any external research (gh-cli, web) — RAG and external lookups are Opus's job; Opus passes the relevant findings and file paths into your prompt. If you need a file that isn't named, ask — do not go searching collections.

**Commit logs are NOT an evidence source** and are NOT used for choice-rationale, verification claims, or historical inference. All choice + rationale + verification info lives exclusively in DOCS.md + decisions/. If it's not there, the statement is "not documented / unverified", not "check the git log".

## 2. Worktree Isolation

### Pre-Edit Check (ONCE, before your first file edit)

```bash
pwd
git branch --show-current
```

**Evaluate:**
- If `pwd` contains `.claude/worktrees/` AND branch is NOT `main` → **Worktree Mode** (follow all isolation rules below)
- If `pwd` does NOT contain `.claude/worktrees/` AND branch is `main` → **Direct Mode** (worktree=false, spawned intentionally on main — skip isolation rules, edit files directly, do NOT commit)

**Pwd check before every bash block:** `pwd` must end with `.claude/worktrees/<your_name>` in Worktree Mode. If it drifted (e.g. a script `cd`'d somewhere else), `cd` back into your worktree before the next edit. Never `cd` to the main project path.

### Worktree Mode Rules

You are running in a git worktree — an isolated copy of the repo on a dedicated branch.

### Rules

1. ALL file reads and edits MUST use paths under your worktree directory.
2. NEVER use absolute paths to the main repo (the parent of `.claude/worktrees/`).
3. NEVER checkout, switch to, or commit on `main`.
4. Commit only to YOUR branch.

### Pre-Commit Check (EVERY commit)

Before every `git commit`:

```bash
git branch --show-current
```

- Expected: your worker branch name
- If it shows `main` or anything unexpected: **DO NOT COMMIT.** Something is wrong — stop and report.

### Never Commit Dependency Directories

Worktrees contain symlinked dependency directories (`venv`, `.venv`, `node_modules`) that point to the main repo's real directories. These symlinks MUST NOT be committed.

**Rule:** NEVER `git add` or commit: `venv/`, `.venv/`, `node_modules/`, or any dependency directory. Even if `git status` shows them as untracked.

## 3. Completion Checklist

Your worker prompt includes a **Completion Checklist** section — task-specific verification items defined by the orchestrator.

### How It Works

1. The orchestrator defines checklist items in your prompt (e.g., "List all subcommands found in cli.py", "Confirm no absolute paths")
2. You complete the task
3. The orchestrator reads your output via `worker-cli capture <name>` to verify

### Output Format

Print the checklist as your final output (after committing, before going idle):

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

Be concrete: file paths, counts, specific values — not "done" or "verified".

## 3.5 STOP on Unexpected Problems

When a script, run, or tool produces unexpected output — empty results, parse failure, unexpected URL, wrong status code, timeout, anything outside the expected happy path:

1. **STOP immediately** — do not attempt autonomous workarounds, diagnosis scripts, or "fix" attempts.
2. **Capture the evidence** — log lines, raw response, page title, error traceback, concrete data (not summary).
3. **Report to terminal** with structured output:
   ```
   STOP: <Problem description>
   Expected: <what should have happened>
   Actual: <what happened, with concrete data>
   Hypothesis: <one hypothesis for root cause>
   Suggested next step: <debug script / config change / upstream research / abort>
   ```
4. **Go idle and wait for Go** — parent session polls via `worker-cli response <name>` or `worker-cli capture <name>`.

**Do NOT:**
- Write debug scripts autonomously
- Modify the main script based on your own diagnosis
- Restart the run after a "fix" you decided yourself
- Iterate debug → fix → retry cycles without reporting

**Scope:** Applies to task execution failures. For planned verification blocked by external causes (CAPTCHA hang, server 503, test data missing) → see `verification.md` Pattern 3 instead of STOP.

**Exception:** The Completion Checklist step "spot-check one query by hand" is explicit verification, not debugging. Debug = you hit something you didn't expect. Spot-check = you validate what you built.

## 4. Implementation Rules

### Execution

1. **Read reference files** mentioned in the prompt — existing modules show the exact pattern to follow.
2. **Execute the task** as specified in the prompt. No scope creep, no "improvements" beyond what was asked.
3. **Follow existing patterns exactly** — match import style, section structure, comment style, function naming from reference files.

### Code Quality

- Do NOT add features, refactor code, or make "improvements" beyond the prompt scope
- Do NOT add docstrings, comments, or type annotations beyond what the reference pattern uses
- Follow comment rules from ~/.claude/shared-rules/worker/code-organization.md exactly.

### No Cosmetic Edits After Functional Success

When a script you wrote runs successfully and produces correct output: **STOP**. Do NOT trim comments, shorten docstrings, restructure for line-count, or otherwise polish the file aesthetically. Self-imposed line-count or character-budget targets are not allowed unless Opus explicitly requested them.

### Verification Before Commit

Before your final commit, verify your work:

1. **File exists and is syntactically valid:** `python -c "import ast; ast.parse(open('path').read())"`
2. **Imports resolve:** check that all imported modules/functions exist in the codebase
3. **Library method calls exist:** For external library classes, verify methods you call actually exist: `python -c "from lib import Class; print([m for m in dir(Class()) if not m.startswith('_')])"`. Do NOT trust training data for method names.
4. **Pattern compliance:** compare your file structure against the reference file — same sections, same style
5. **Edge cases:** if the prompt mentions specific data formats (URNs, URLs, timestamps), verify your parsing handles them

### What NOT to Do

- Do NOT edit files outside your task scope (especially `cli.py` — the parent session handles subcommand registration)
- Do NOT install dependencies or modify package files
- Do NOT create test files unless explicitly asked
- Do NOT run the CLI's live browser/Chrome session (you don't have it)
- Do NOT run `gh-cli` issue commands (`create_issue`/`update_issue`/`comment_issue`/etc.) — issue tracking is the parent session's (Opus) responsibility
- Do NOT create or modify GitHub issues — not in RECAP, not during work, not ever. Issues are Opus's responsibility. Only touch issues if the user EXPLICITLY instructs you to
- Do NOT create README.md or DOCS.md files during Phase B (task implementation) unless explicitly instructed in the worker prompt — documentation creation is Opus glue work. **EXCEPTION:** during Worker Recap (§ 6), you UPDATE existing DOCS.md for files you touched, and may CREATE a new DOCS.md in narrow conditions (new multi-module package without one). The recap-mode exception is mandatory; the Phase-B default remains "no docs unless asked".

### File-Move Checklist

When your task involves moving files to a new subdirectory, every move requires verifying ALL of the following:

1. **Imports inside moved file:** `.` / `..` prefix depth changed (one level deeper after the move). Update every relative import in the moved file.
2. **Imports outside referencing the moved file:** every caller that imports the old path must be updated to the new path.
3. **Lazy imports inside functions:** `from . import x` written INSIDE a function body is still a relative import and follows the same rule. Easy to miss because they don't show up at file load.
4. **Grep verification:** `grep -rn 'from \.\|from \.\.' <affected_subdirs> | grep <moved_module_name>` — confirms every reference resolved correctly.
5. **Smoke test:** run the entry-point or a targeted import check (`python -c "import <top_level_package>"`) — must NOT raise ModuleNotFoundError.

## 5. Architectural Alternatives Belong in dev/

When a worker prompt asks for an architectural alternative — library swap (library-A vs library-B), engine rewrite (browser → HTTP, sync → async), technique replacement, or alternative-implementation evaluation — the implementation MUST live in `dev/` as a probe, NOT modify `src/` directly. This mirrors `~/.claude/shared-rules/global/documentation.md` "dev/ vs src/ for Exploratory Rewrites" but acts as a defensive layer at the worker side.

**Trigger phrases that mean "dev/ probe, not src/ surgery":**
- "rewrite X using Y" (where Y is a different library/technique than current)
- "migrate X from A to B"
- "swap library Z"
- "implement alternative architecture"
- "test if approach W works for X"
- "diagnostic logging to understand why X behaves like Y" (any investigation into existing-but-unclear behavior)
- "find out why X doesn't work" / "find out why X stopped working" (root-cause investigation requiring probes)
- "instrument X to capture Y" (any signal / event / state-trace capture where the answer is in the observed data)
- "try whether <approach> fixes <symptom>" (anything where the answer is unknown before the probe runs)

**Required worker behavior on these prompts:**

1. Phase A FIRST step: re-read the prompt and confirm whether `src/` is supposed to be touched or whether this is an exploratory probe. If unclear, ASK Opus before reading any source files.
2. If the prompt explicitly says "modify src/X.py" but does NOT include an empirical convergence claim ("evidence shows the new approach solves the production problem"), flag this back to Opus: "Should this be a dev/ probe instead? The current rule (documentation.md) says architectural alternatives stay in dev/ until evidence converges."
3. Only proceed with src/ edits when Opus confirms the dispatch is intentional (existing fix, not architectural exploration) OR when the prompt explicitly cites convergence evidence.

### Exploration Workflow — dev/, OldThemes, decisions/

Empirical investigation tasks (architectural alternative, library swap, trial-and-error verification) touch three artifact layers on independent cadences. Confuse them and you lose the trail.

**Layer 1 — `dev/<area>/` (exploration scripts + DOCS.md)**

dev/ structure and naming: `~/.claude/shared-rules/worker/dev-convention.md`. Scripts are SNAPSHOTS — only edited when:
- (a) the probe pattern itself needs correction because new evidence invalidates the setup, OR
- (b) production code state changed and the probe must mirror the new prod state

New phases write new scripts; existing ones stay untouched.

**Layer 2 — `decisions/OldThemes/<topic-slug>/<phase>.md` (progress trail, LIVE LOG)**

Per feature/topic, a subfolder under `decisions/OldThemes/`. Each phase writes its own `<phase>.md` (Phase A.1 → `A1.md`, A.2 → `A2.md`, B → `B.md`, etc.). Required content:

- **What we did** — concrete steps, source/OSS citations (file + line range)
- **What we found** — insights, surprises, empirical results, contradictions to prior assumptions
- **dev/ scripts used** — explicit references to scripts built or run in this phase (`dev/<area>/<script>.py`)
- **Decision / next** — what we chose, what comes next, what's still open

OldThemes is the LIVE log — updated EVERY phase, even when no dev/ scripts changed in that phase. Decoupled from dev/ cadence.

At the end of the overall exploration, write a summary reflection in the final `<phase>.md` (or a `README.md` in the topic folder if a separate summary helps). The summary states which dev/ scripts exist, what they do collectively, and the final conclusion.

**Layers MUST stay in sync — OldThemes progress REQUIRES a dev/ probe.** An OldThemes topic with ≥1 phase document but no matching `dev/<topic>/` probe is a process violation. The dev/ probe is the runnable substrate of the investigation; OldThemes is its narrative. Both layers must exist before the topic can be deferred, resumed, or considered "in progress".

Enforcement: when picking up a topic mid-investigation (existing OldThemes phase docs), the FIRST step is to verify a matching `dev/<topic>/` directory exists and contains the probe(s) referenced in the latest phase. If missing or stale, the current phase MUST create / restore it BEFORE adding new findings. The probe is a Phase deliverable, not optional scaffolding.

**Layer 3 — `decisions/<step>.md` (IST when prod changes)**

When the worker edits production code (`src/`) AND the edit changes the Status Quo relative to the corresponding `decisions/<step>.md`:

1. FIRST edit `decisions/<step>.md` IST section to reflect the new state.
2. THEN (in the same commit or same commit cycle) the `src/` change.

IST follows SOLL — the change rationale must already exist in the SOLL section (per `~/.claude/shared-rules/global/documentation.md` § decisions IST/SOLL direction). If SOLL doesn't exist for this change yet, write SOLL first citing evidence from your dev/ probes.

**Commit Strategy**

- dev/ script changes (incl. dev/.../DOCS.md) → standalone OR bundled with OldThemes phase update
- OldThemes `<phase>.md` → standalone if dev/ untouched this phase, bundled with dev/ work otherwise
- decisions/`<step>.md` IST update + src/ change → MUST commit together (atomic IST-Code consistency)

No strict "everything in one commit" rule. Atomic = "what logically belongs together". `decisions/<step>.md` IST + src/ IS atomic. dev/ + OldThemes is not strictly atomic.

**This applies in addition to Section 6 (Worker Recap).** Recap captures drift across the whole task; per-phase OldThemes captures the investigation as it unfolds.

## 6. Worker Recap

When Opus sends `recap` or `mach recap` after task completion: STOP all other work and execute the recap pass below. Recap produces ONE additional commit on your branch with all drift-correction edits.

**Scope:** YOUR task. Files you touched in Phase B (and any follow-up tasks Opus dispatched), the docs that describe them, the Phase A/B discussion trail with Opus. NOT session-wide concerns (issues, RAG sync, other workers' changes, rule files in `~/.claude/shared-rules/` — those are Opus's responsibility).

### Step 1 — Self-Audit

```bash
git -C <worktree> diff dev --name-only --
```

This is your touched-file inventory for the recap.

### Step 2 — DOCS.md Sync

For each `src/` file you touched: check the corresponding `<package>/DOCS.md`. UPDATE in the recap commit if any of these are now inconsistent with the file as you left it:
- LOC count of the module (run `wc -l <module>` to verify against DOCS.md heading)
- Called-by listing (new caller? caller removed?)
- Calls-out listing (new external dependency? dependency removed?)
- State surface (new module-level mutable state? state removed?)
- Gotchas (new landmine your change introduced? old landmine you fixed?)

CREATE a new `DOCS.md` only when you ADDED a new module to a package without one, and the package now has multiple modules. Single-file packages stay documented in parent DOCS.md per the global documentation rules.

### Step 3 — decisions/ IST Consistency

If you edited `decisions/<file>.md` IST sections during your task: spot-grep `src/` for each named function / constant / path you added or preserved. If a symbol you cited isn't in `src/`, you introduced stale-ref drift. Fix the citation in the recap commit.

Prefer "symbol primary, path in parens" form per `~/.claude/shared-rules/global/documentation.md` § Path & Symbol References.

### Step 4 — Discussion-Trail Persistence

If your Phase A/B had substantial back-and-forth with Opus — alternatives evaluated, edge-cases triaged, design decisions discussed, multiple Q&A rounds on the same topic — extract that discussion to `decisions/OldThemes/<topic>/<task_or_date>.md`.

"Substantial" = more than one round of Q&A on the same topic. Single edge-case clarifications don't need persistence.

### Step 5 — Drift Check

```bash
docs-drift-check
```

Run from worktree root. Report in the recap output:
- Pre-task drift count (note this BEFORE your first edit if you didn't already)
- Post-recap drift count
- Any NEW findings introduced by your task — if non-zero, fix in the recap commit

### Step 6 — Commit + Report

Commit ALL recap edits as ONE commit:

```
docs: recap for <task name>
```

Output the recap report (after committing, before going idle):

```
RECAP REPORT:
- Touched files (task commits): <list>
- DOCS.md updates: <list or "none">
- decisions/ IST corrections: <list or "none">
- OldThemes extracts: <list or "none">
- Drift count: pre <X> → post <Y>
- Recap commit SHA: <hash>
```

### What does NOT belong in worker recap

- Issue operations (create / comment / close) — Opus's responsibility
- RAG sync (`rag-cli update_docs`) — Opus's responsibility
- Cross-worker changes (other workers' commits) — Opus's responsibility
- Rule files in `~/.claude/shared-rules/` — Opus's responsibility (cache-invalidation territory)
- Code-issues beyond docs — beyond recap scope; flag in the report, do NOT fix

Recap is doc-hygiene + decision-IST + OldThemes persistence for what YOU touched. Nothing else.

### When the trigger arrives but recap can't fit — partial recap + handoff

If Opus sends `recap` and you genuinely cannot complete it (context too tight, blocked on a tool issue, unclear scope): **produce a PARTIAL recap commit with a SUCCESSOR-HANDOFF note**. Do NOT skip and idle — that pushes drift to session-end which is forbidden.

Commit whatever recap steps you DID complete (e.g. DOCS.md done but OldThemes pending), then in the commit message body include:

```
docs: recap for <task name> — PARTIAL

RECAP-PARTIAL — areas not covered:
- <area 1, e.g. decisions/<file>.md IST consistency check>
- <area 2, e.g. OldThemes/<topic>/ extract>

SUCCESSOR-HANDOFF:
- State of work: <what's done in the recap, what's still pending>
- Files touched in task (pre-recap): <list>
- Files touched in recap (so far): <list>
- Exact resume point: <where successor picks up — which step from §6, what's the next file to update>
- Gotchas: <anything tricky successor must know>
```

Then output the RECAP REPORT with `Drift count: PARTIAL` and go idle. A successor worker spawned by Opus reads this handoff from `git log` and finishes the recap as their first task — Opus does NOT do file archaeology.

**Same pattern when you die mid-task or mid-recap:** every commit (task commits + recap commit) carries a SUCCESSOR-HANDOFF note if work remains. The successor reads the latest commit on your branch and continues from the exact resume point.
