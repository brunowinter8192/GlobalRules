
# Worker Rules — Worktree Isolation & Report

These rules apply to every worker session. Run the Pre-Edit Check to determine your mode.

## 1. Code Investigation — Docs + decisions via RAG

Bei jeder Frage zur Beschaffenheit des Codes — warum ist X so, ist Y verifiziert, was war die Wahl-Begründung, wie wird Z verwendet — zuerst `rag-cli search_hybrid "<query>" <Project>-meta` absetzen. Die Meta-Collection indexiert DOCS.md (auch unter dev/), decisions/, CLAUDE.md. Wenn der Context existiert, findet RAG ihn. Code-Read kommt danach für Detail.

**Commit-Logs sind keine Evidenz-Quelle und werden NICHT für Wahl-Begründungen, Verifikations-Aussagen oder historische Inferenzen herangezogen.** Sie sind absichtlich kurz gehalten — dokumentieren Was, nicht Warum. Alle Info zu Wahl + Begründung + Verifikation lebt ausschließlich in DOCS.md + decisions/. Wenn dort nichts steht: Aussage ist "nicht dokumentiert / unverifiziert", nicht "im git log nachschauen".

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

Worktrees contain symlinked dependency directories (`venv`, `.venv`, `node_modules`) that point to the main repo's real directories. These symlinks MUST NOT be committed — they become circular self-references when merged back.

**Rule:** NEVER `git add` or commit: `venv/`, `.venv/`, `node_modules/`, or any dependency directory. Even if `git status` shows them as untracked.

## 3. Completion Checklist (MANDATORY)

Your worker prompt includes a **Completion Checklist** section — task-specific verification items defined by the orchestrator.

### How It Works

1. The orchestrator defines checklist items in your prompt (e.g., "List all MCP tools found in server.py", "Confirm no absolute paths")
2. You complete the task
3. **Before your final commit**, output the filled checklist to the terminal — this is your last action
4. The orchestrator reads your output via `worker_capture(tail=N)` to verify

### Output Format

Print the checklist as your final output (after committing, before going idle):

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

Be concrete: file paths, counts, specific values — not "done" or "verified".

### STOP on Unexpected Problems (NON-NEGOTIABLE)

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
4. **Go idle and wait for Go** — parent session is automatically notified and reads via `worker_capture`.

**Do NOT:**
- Write debug scripts autonomously
- Modify the main script based on your own diagnosis
- Restart the run after a "fix" you decided yourself
- Iterate debug → fix → retry cycles without reporting

**Why:** Opus has user-facing context (session goal, scope, time budget, prior decisions). A single autonomous fix cycle can invalidate the entire session approach. Your job is to document the problem clearly, not to fix it.

**Exception:** The Completion Checklist step "spot-check one query by hand" is explicit verification, not debugging. Debug = you hit something you didn't expect. Spot-check = you validate what you built.

## 4. Implementation Rules

### Execution

1. **Read project CLAUDE.md first** — it contains module patterns, naming conventions, and coding rules specific to this project.
2. **Read reference files** mentioned in the prompt — existing modules show the exact pattern to follow.
3. **Execute the task** as specified in the prompt. No scope creep, no "improvements" beyond what was asked.
4. **Follow existing patterns exactly** — match import style, section structure, comment style, function naming from reference files.

### Code Quality

- Do NOT add features, refactor code, or make "improvements" beyond the prompt scope
- Do NOT add docstrings, comments, or type annotations beyond what the reference pattern uses
- Header comments on functions: one line describing WHAT, matching reference file style
- NO comments inside function bodies unless the reference pattern has them
- Match the exact section structure of reference files (e.g., INFRASTRUCTURE / ORCHESTRATOR / FUNCTIONS)

### No Cosmetic Edits After Functional Success

When a script you wrote runs successfully and produces correct output: **STOP**. Do NOT trim comments, shorten docstrings, restructure for line-count, or otherwise polish the file aesthetically. Self-imposed line-count or character-budget targets are not allowed unless Opus explicitly requested them.

The signal "the script ran, the output is correct" → next action is COMMIT, not polish. A script that has been edited to trim 4-character comments down to 3-character comments is wasting context budget on no behavioral change. Once functional, capture-and-commit; let format-pedantry happen in a follow-up if needed.

### Verification Before Commit

Before your final commit, verify your work:

1. **File exists and is syntactically valid:** `python -c "import ast; ast.parse(open('path').read())"`
2. **Imports resolve:** check that all imported modules/functions exist in the codebase
3. **Library method calls exist:** For external library classes, verify methods you call actually exist: `python -c "from lib import Class; print([m for m in dir(Class()) if not m.startswith('_')])"`. Do NOT trust training data for method names — libraries change APIs between versions.
4. **Pattern compliance:** compare your file structure against the reference file — same sections, same style
5. **Edge cases:** if the prompt mentions specific data formats (URNs, URLs, timestamps), verify your parsing handles them

### What NOT to Do

- Do NOT edit files outside your task scope (especially `server.py` — the parent session handles tool registration)
- Do NOT install dependencies or modify package files
- Do NOT create test files unless explicitly asked
- Do NOT run the MCP server or make MCP tool calls (you don't have the Chrome session)
- Do NOT run `bd` commands (bead CLI) — worktrees copy `.beads/` state, and bd operations corrupt the main repo's bead data
- Do NOT create beads (via MCP tools or CLI) — not in RECAP, not during work, not ever. Beads are the parent session's (Opus) responsibility. Only create beads if the user EXPLICITLY instructs you to
- Do NOT create README.md or DOCS.md files during Phase B (task implementation) unless explicitly instructed in the worker prompt — documentation creation is Opus glue work. **EXCEPTION:** during Worker Recap (§ 6), you UPDATE existing DOCS.md for files you touched, and may CREATE a new DOCS.md in narrow conditions (new multi-module package without one). The recap-mode exception is mandatory; the Phase-B default remains "no docs unless asked".

### File-Move Checklist

When your task involves moving files to a new subdirectory, every move requires verifying ALL of the following:

1. **Imports inside moved file:** `.` / `..` prefix depth changed (one level deeper after the move). Update every relative import in the moved file.
2. **Imports outside referencing the moved file:** every caller that imports the old path must be updated to the new path.
3. **Lazy imports inside functions:** `from . import x` written INSIDE a function body is still a relative import and follows the same rule. Easy to miss because they don't show up at file load.
4. **Grep verification:** `grep -rn 'from \.\|from \.\.' <affected_subdirs> | grep <moved_module_name>` — confirms every reference resolved correctly.
5. **Smoke test:** run the entry-point or a targeted import check (`python -c "import <top_level_package>"`) — must NOT raise ModuleNotFoundError.

## 5. Architectural Alternatives Belong in dev/

When a worker prompt asks for an architectural alternative — library swap (httpx vs pydoll, requests vs httpx), engine rewrite (browser → HTTP, sync → async), technique replacement, or alternative-implementation evaluation — the implementation MUST live in `dev/` as a probe, NOT modify `src/` directly. This mirrors `~/.claude/shared-rules/global/documentation.md` "dev/ vs src/ for Exploratory Rewrites" but acts as a defensive layer at the worker side.

**Trigger phrases that mean "dev/ probe, not src/ surgery":**
- "rewrite X using Y" (where Y is a different library/technique than current)
- "migrate X from A to B"
- "swap library Z"
- "implement alternative architecture"
- "test if approach W works for X"

**Required worker behavior on these prompts:**

1. Phase A FIRST step: re-read the prompt and confirm whether `src/` is supposed to be touched or whether this is an exploratory probe. If unclear, ASK Opus before reading any source files.
2. If the prompt explicitly says "modify src/X.py" but does NOT include an empirical convergence claim ("evidence shows the new approach solves the production problem"), flag this back to Opus: "Should this be a dev/ probe instead? The current rule (documentation.md) says architectural alternatives stay in dev/ until evidence converges."
3. Only proceed with src/ edits when Opus confirms the dispatch is intentional (existing fix, not architectural exploration) OR when the prompt explicitly cites convergence evidence.

**Why the defensive layer:** Opus may dispatch a src/-modifying prompt for an architectural alternative without realizing the rule applies (happened 2026-05-08 with the searxng Scholar HTTP migration — went directly into src/, smoke showed incomplete fix, work was discarded but the empirical proof would have been preserved had it lived in dev/). The worker-side check catches the dispatch before the wasted work begins.

## 6. Worker Recap (MANDATORY when triggered)

When Opus sends `recap` or `mach recap` after task completion: STOP all other work and execute the recap pass below. Recap produces ONE additional commit on your branch with all drift-correction edits.

**Scope:** YOUR task. Files you touched in Phase B (and any follow-up tasks Opus dispatched), the docs that describe them, the Phase A/B discussion trail with Opus. NOT session-wide concerns (beads, RAG sync, other workers' changes, rule files in `~/.claude/shared-rules/` — those are Opus's responsibility).

**Why recap exists:** you have intimate context about WHAT you changed. Opus operates at higher altitude and misses per-task details — LOC drift after a code edit, DOCS.md call-graph changes, decisions/ symbol citations that became stale, design discussion that should land in OldThemes. Per-task recap catches drift at the source.

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

If your Phase A/B had substantial back-and-forth with Opus — alternatives evaluated, edge-cases triaged, design decisions discussed, multiple Q&A rounds on the same topic — extract that discussion to `decisions/OldThemes/<topic>/<task_or_date>.md`. The conversation history is otherwise lost on session end.

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

- Bead operations (create / comment / close) — Opus's responsibility
- RAG sync (`rag-cli update_docs`) — Opus's responsibility
- Cross-worker changes (other workers' commits) — Opus's responsibility
- Rule files in `~/.claude/shared-rules/` — Opus's responsibility (cache-bruch territory)
- Code-issues beyond docs — beyond recap scope; flag in the report, do NOT fix

Recap is doc-hygiene + decision-IST + OldThemes persistence for what YOU touched. Nothing else.

### When the trigger arrives but recap can't fit

If Opus sends `recap` and you genuinely cannot complete it (context too tight, blocked on a tool issue, unclear scope), output:

```
RECAP SKIPPED: <reason>
```

Then go idle. Opus's session-end Recap will absorb the drift cleanup. Do NOT half-commit a partial recap.
