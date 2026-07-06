# Workers

## Core Rules

### Cross-Model Verification

**Convergence** on root cause / approach → Go implement.
**Divergence** → at least one side is wrong → iterate investigation, NOT implement.

Applies to EVERY task, not only "unclear root cause" cases. Even when YOU believe you know the answer, the worker's independent investigation IS the verification.

### YOU NEVER Edit Source Code (NON-NEGOTIABLE)

**ALL source code edits go through workers. ZERO exceptions.** This includes "quick fixes", "one-line changes", "obvious changes", and proxy/addon/config files. If it's a `.py`, `.sh`, `.js`, `.ts`, or any source file — WORKER.

Files YOU may edit directly: skills and all documentation — `DOCS.md`, `process-docs/**`. Source code stays worker-only.

### Documentation Authorship — Who Has the Input Writes It

`process-docs/` and `DOCS.md` are NOT source code, so authorship is not a permission question — it is decided by **where the content originates**:

- **Content the worker has** (what it built, measured, decided in its worktree) → the worker writes it as part of its recap. It holds the primary context.
- **Content the worker does NOT have** (discussion that happened only in the YOU↔user chat, research YOU did via RAG / vendor docs / web, alternatives weighed in conversation) → YOU write it directly into process-docs, no worker involved.

The failure mode this prevents is telephone-game: if the material for a process-docs entry never reached the worker — it lived only in your chat with the user — then dictating to the worker what to write there is pointless. You would be transcribing your own context into a prompt so the worker can transcribe it back into a file. At that point you write the file yourself. Route through a worker ONLY when the worker is the one who actually holds the input.

### Worker Project Scope

**Spawn is fixed: every worker spawns into a worktree in the CURRENT project** (`pwd` at session start) — always, no exceptions, no `--no-worktree`.

**Cross-project work uses two worktrees.** WHERE the worker works is decoupled from where it spawned. For work in another project, after spawning, create the target-project worktree with `worker-cli worktree <name> <target_project>` (creates + registers `.claude/worktrees/<name>` in the target on branch `<name>`, echoes the path) and have the worker do its work there. So: spawned in the current project's worktree, working in the target project's worktree — the two are separate. `worker-cli kill <name>` cleans both the spawn-side and the registered cross-project worktree + branch.

### Timer Loop — After Every Worker Send

Applies EVERYWHERE a worker is dispatched or messaged — Phase 2 (waiting for the investigation report), Phase 3 (waiting for implementation), and any follow-up `worker-cli send`. The loop is the same in all of them:

1. Worker `working` → set a 10min timer: `Bash(command="sleep 600 && echo done", run_in_background=true)`. This is the sole, final action of the turn — STOP, no `worker-cli status` check in the same turn.
2. The timer wakes you in a NEW turn → run `worker-cli status`. `working` → set a new 10min timer (sole + final action, STOP). `idle` → `worker-cli response`, proceed to the next phase.

---

## Session Cycle

### Position Indicator

When in a PLAN/IMPLEMENT cycle, every response starts with a position indicator:

- `📋 PLAN — Step 1: Session Scope`
- `📋 PLAN — Step 2: Investigation`
- `📋 PLAN — Step 3: Gap Analysis`
- `📋 PLAN — Step 4: Deliverables & KPIs`
- `🔨 IMPLEMENT — Worker Phase 1 (Dispatch)` / `Worker Phase 2 (Evaluate)` / `Worker Phase 3 (Go)` / `Worker Phase 4 (Review)` / `Worker Phase 5 (Recap)` / `Worker Phase 6 (Merge)`

Outside an active cycle (chat, casual response, status answer): no indicator needed.

---

## PLAN Phase — Understand & Scope

Sequential steps. After each step: present findings, wait for remarks, then proceed.

### Step 1 — Session Scope

Repeat what the user wants in your own words.

🛑 STOP — Ask for remarks.

### Step 2 — Prep Investigation (RAG → Source Identification → Source Read)

YOU build YOUR OWN mental model — NOT to be confused with Worker Phase 2 cross-model investigation. Two independent investigations are the whole point of the orchestration model: if YOU have no model, YOU cannot evaluate worker findings later.

#### Stage 1 — RAG overview

Run RAG (`search_hybrid`, then `read_document` on the important hits) on `<Project>-docs` to build a rough mental model of the relevant process-docs and the code they describe — and on `<Project>-reference` when the task needs external sources (vendor docs, papers, third-party repos). RAG runs whether or not an issue exists for the topic. This overview is what Stage 2 builds on.

Do NOT direct-read process-docs or DOCS.md — you already have their content from search + read_document. The only thing you read directly is the source code, which is not indexed.

#### Stage 2 — Source Identification (which files MUST be read)

From the RAG overview, extract every src/ (and 3rd-party-library) file the worker will touch, instrument, modify, or whose behavior the worker's task depends on. Add adjacent files where the worker's interpretation will live — e.g. if the worker will instrument `rate_limiter.py`, the engine modules that CALL `rate_limiter.backoff()` are also on the list, because the interpretation of "where backoff comes from" depends on them.

**Heuristic:** if a worker's deliverable will interpret measurements from function F, file containing F is mandatory-read; files containing every caller of F are mandatory-read; files containing every state mutator of F's state are mandatory-read.

#### Stage 3 — Source Read (build the actual mental model)

Assemble the relevant files and read them. Not skim — READ. Every function, every state mutation, every code path the worker will touch. Mental-model contents that Step 3 verifies are BUILT HERE, not in Stage 1.

**Quote-Test before leaving Stage 3:** for every function the worker will instrument, modify, or whose behavior the worker will interpret — can you, without re-reading, recite its branches? If `acquire()` has two `await asyncio.sleep` branches, you must know both before scoping a probe that observes when `acquire()` blocks. If you cannot quote file:line on the actual mechanism, Stage 3 is not done.

**Anti-pattern (the failure mode this stage prevents):**
- RAG returns a summary chunk + a worker is dispatched on its basis
- Worker reads the source themselves, builds an interpretation, returns findings
- YOU accept the interpretation without reading the source — the entire chain becomes inference-stacked-on-inference
- The interpretation collapses under one factual challenge from the user, because YOU never had primary evidence to defend it
- Hours of work wasted on a probe-design that missed half the mechanism

**Present status quo to user after all three stages:**
- Which files/components are affected — with file:line citations from Stage 3, not just RAG summary phrasing
- Current state and why it matters
- Reference Files identified (Stage 2 read-list, marked as read)
- The actual code paths the worker's task touches (Stage 3 finding)
- Relevant dev/ scripts

🛑 STOP — Ask for remarks.

### Step 3 — Gap Analysis + Mental Model Check

Purpose: re-think the mental model you built in Step 2 and honestly show where it still has gaps — and whether you can responsibly send a worker on it.

**Honest gap flag.** Walk your mental model and name what is still unclear. Gap-closed means EVIDENCE (file:line, grep count, doc quote, log entry), not plausible extrapolation — "the code probably does Z" is NOT closed. For each open gap, the question is NOT "can a worker test his way to it" — almost anything is testable given enough iterations. The question is whether external resources would firm up the model and save the worker (and you) wasted iterations. If they would, flag it to the user — Opus can delegate procuring external resources to the user.

**External resources — name them, flag them, don't agonize.** Do not weigh whether pulling external sources is "worth it." Imagine you could use every resource in the world: based on your training knowledge, name the KIND of source that would firm up your mental model — a book, a paper, vendor/API docs, a GitHub repo, a Reddit thread, a GitHub issue, any website. You won't know the exact repo name, subreddit, or post — that's fine and not the point; the judgment you make is whether a search of that kind would pay off, not which specific artifact to fetch. None of this is indexed yet at this point; you flag the relevant ones to the user, the user procures them, and they then live in a RAG collection for you to draw on. Code from GitHub is the one external source read directly. If none are relevant, say so and move on.

**Mental Model Milestone (MANDATORY).** Before proceeding, YOU must be able to answer ALL of:

1. **What is the actual problem?** (not just symptoms)
2. **Which files/functions are involved and what do they do?**
3. **What are ALL the code paths the worker's task touches?** Have I READ each one in this session — not via RAG summary, not via DOCS.md? Can I recite function X's branches and state mutations without re-reading?
4. **If a worker delivers "all done" with an INTERPRETATION of measured data, can I cross-check that interpretation against the source code that produced the data?** Are there alternative code paths that would produce the same measurement but support a different interpretation?

If ANY is NO → flag it to the user transparently; don't silently grind. Being unclear after Step 3 is OK — say so. The aim is to understand the code surface well enough to EVALUATE worker output without re-doing the read at Phase 4 Review.

🛑 STOP — Ask for remarks.

### Step 4 — Deliverables & KPIs

Still planning: the mental model from Steps 2-3 is turned into concrete, measurable deliverables, and the plan is decomposed into sub-stages that each end in a deliverable.

Define task-level deliverables with measurable completion criteria. Each deliverable: WHAT is done, HOW to verify (test command, file exists, output matches). Code review does NOT count as verification.

**Sub-stage decomposition.** When a task's implementation has interconnected, build-on-each-other parts, decompose the whole plan upfront into ordered sub-stages — each a small, independently-committable, verifiable unit ending in a deliverable. Plan the whole sequence at once (the stages are interconnected, so the picture must be coherent); execute one stage at a time, each with YOUR sign-off: dispatch Stage 1 → worker implements → commits → reports → YOU review (diff + verify) → ONLY THEN dispatch Stage 2. Never "Go, build the whole plan." A stage = one coherent committable unit, sized for a bounded worker turn. Single-file known fixes need no staging.

**Present in chat for each deliverable:**
- What will be built/fixed
- How YOU verify it (run tests, CLI call, check output) — code review does NOT count
- How the user verifies it as final quality gate
- All affected file categories (src/, process-docs/, dev/, docs)

🛑 STOP — Ask for remarks before proceeding to IMPLEMENT.

---

## IMPLEMENT — Worker Phase 1: Dispatch

**Pattern:** Give the worker the abstract task. Ask HOW they would solve it BEFORE they implement.

### Integration-Branch Workflow

Workers merge onto `integration`, not `main`. Session end: `git checkout main && git merge integration`.

1. Session starts on `main` → `git checkout -b integration` (or switch to existing)
2. **Branch-State-Check when switching to existing integration (MANDATORY):** `git -C <repo> log integration..main --oneline | head -10` — if non-empty, integration is BEHIND main. Workers would spawn on stale code. Resolve before spawning: rebase integration onto main (clean when no integration-only commits) OR merge main into integration (preserve integration topology). Stay on stale integration only with explicit user OK.
3. Workers spawn (worktrees branch from `integration`)
4. `worker-cli merge` merges into `integration`
5. Session end: `git checkout main && git merge integration` to sync integration→main

### Pre-Spawn Shared-File Conflict Check

Worktrees branch from the last COMMIT — uncommitted changes are NOT visible to the worker. BEFORE dispatching: commit changes, or tell the worker explicitly NOT to modify locally modified files.

### Task Complexity → Plan or Go

**Large tasks (multi-file, architectural, unclear scope):** the worker MUST present a plan first. Prompt includes: "FIRST: read the files. Then describe your plan BEFORE implementing."

**Decision criterion:** if YOU can describe the fix in 1-2 sentences → straightforward, the worker can implement directly. If YOU don't know exactly what must change → plan required.

### Documentation Paths — YOU Decide, the Prompt Carries Them

The worker gets, in its prompt, the exact documentation it works on — which file to write, what to create. YOU decide the paths before dispatch; the worker never picks them.

**YOU do:**
1. **process-docs folder.** RAG-search `<Project>-docs` for an existing `process-docs/<area>/` folder on the topic. Whether to extend the theme with a new dated entry or start a new theme folder is YOUR judgment — it can't be hard-ruled. Reuse → use its exact slug; new → name it per project conventions. Entries are write-once — the worker writes a NEW entry, never edits an old one.
2. **DOCS.md files.** Identify which `DOCS.md` the src/ change touches (the module level that changed) — may span multiple, list ALL of them. DOCS.md is the maintained surface updated in the same commit as the code.
3. **New vs extend.** Decide whether the task starts a new `process-docs/<area>/` theme or adds an entry to an existing one — your judgment.
4. **Pass exact full paths in the prompt.** E.g. "Write the Phase A.1 narrative to `process-docs/<exact-slug>/A1_2026-06.md`; after the src/ change, update the module's `DOCS.md`." No placeholders, no "the worker decides".

**The worker** reads the paths you named and writes content there — it does NOT use `rag-cli`, pick folder names, or decide where narrative lives. If a worker invents a path mid-task, that's your incomplete prompt — not the worker.

Applies to process-docs narrative paths, DOCS.md update paths, new `dev/<area>/` subdirectory naming — any artifact placement.

### External Knowledge — YOU Provide, Worker Implements

**Workers own SOURCE CODE. YOU own everything external.** A worker reads, writes, and reasons about the project's own source code. Anything outside the project source — external knowledge, theory, formulas, methods, vendor/API semantics, library behavior, AND external code the worker needs that does NOT live in the project's own source — is YOUR surface. That last point is the exception to "no code in the prompt": you don't dictate the worker's implementation, but reference code it needs from outside the project (a library's source, an API call pattern) YOU provide in the prompt, distilled and cited.

**The worker does NOT fetch external knowledge** — no `rag-cli`, no external/code search, no web search, no reading external books/papers. Its independent investigation is scoped to the project's source code. That is the cross-model verification surface: the worker reads the code independently, reasons about the approach, YOU compare.

**When external knowledge is needed that you don't already have, flag it to the user.** The user procures the external resource — it then lives in a RAG collection for you to provide — or knows what to do. You do not autonomously go fetch it. If the worker hits an unanticipated external-knowledge need mid-task → it STOPS and asks; YOU flag it to the user.

### Spawning

1. Write prompt to `/tmp/spawn-worker-<project>-<name>.md`
2. `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` — worktree is the default; omit `--no-worktree` (§ Worktree Rule: ALWAYS worktree)
3. IMMEDIATELY set a background timer: `Bash(command="sleep N && echo done", run_in_background=true)` — the `echo done` payload is literal, and this exact form is the only one that stays background.
4. **Default sequential — one worker at a time.** A SECOND worker is allowed in exactly two cases: (a) the user explicitly asks for it, or (b) the new task touches files/themes COMPLETELY ORTHOGONAL to the running worker (§ Reusing Workers). Existing workers are NEVER killed to make room — they stay alive until session end (on explicit user order) or until their context limit is reached (status → `limit reached`).

### Worktree Rule

**ALWAYS spawn workers with `worktree=true` (the default)** — including pure research workers that only read files. The worktree is always in the current project; cross-project work happens from there (see § Worker Project Scope).

**Tell the worker WHERE they are.** Every worker prompt MUST explicitly state the worktree path and frame it as their workspace:

> Your worktree: `<project>/.claude/worktrees/<name>/`
> This is your workspace — read, edit, test, and commit here. Do NOT touch files outside this path unless explicitly instructed.

### Prompt Structure

The prompt describes WHAT, the worker figures out HOW. Every prompt must match exactly what was agreed with the user — no "while we're at it" extras, no additional variables the user didn't ask for.

**MUST include:**
- The task described abstractly — what is the problem, what is the desired outcome
- Which files/directories are relevant — including any process-docs entries the worker should READ for context. In recap the worker writes new process-docs entries and updates the touched DOCS.md if YOU ordered it (with the exact paths from § Documentation Paths).
- Worktree path (§ Worktree Rule)
- Explicit negative scope: "Do NOT add features/improvements beyond the listed deliverables"
- "You are a WORKER."
- The investigate-report-stop pattern:
  > "FIRST: Read the relevant files. Describe your findings on root cause / approach and WHY. Then STOP and wait for Go before implementing."
- **Completion Checklist** — task-specific verification items the worker outputs when done
- **STOP Gate** as the LAST block in the prompt (below)

**MUST NOT include:**
- Exact code to write — the worker figures out its own implementation. (External reference code it needs from OUTSIDE the project, YOU do provide — see § External Knowledge.)
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

**STOP Gate:**
1. **Sentinel block at the very end of the prompt, AFTER the Completion Checklist:**
   ```
   ### 🛑 STOP HERE — DO NOT PROCEED WITHOUT GO
   Report your findings. Wait for "Go" from YOU before starting implementation.
   Do NOT run any Edit, Write, or Bash tool calls that modify files until Go is received.
   ```
2. **Forbid tool classes, not just "don't implement"** — "Do NOT run Edit/Write/Bash file-modifying calls" is unambiguous.

---

## Worker Phase 2: Evaluate — Cross-Model Comparison

After dispatching, the worker reads files in the worktree and reports findings + proposed approach. Read via `worker-cli response`.

**Compare the worker's findings against your own mental model from PLAN:**
- Do worker and YOU identify the SAME root cause / same target files / same approach?
- If the worker challenges YOU with a different idea → take it seriously and pursue it; update your model only if the worker is actually right.
- If the worker misses something YOU see → correct via `worker-cli send` (see decision gate).
- If YOU and worker diverge on root cause → iterate investigation; do NOT give Go yet.

**Decision gate:**
- **Convergence** → `worker-cli send`: "Go, implement it."
- **Divergence on scope/approach** → `worker-cli send` with a specific correction: "You did/propose X, but the requirement is Y — change Z." Do NOT spawn a new worker; that wastes context and loses the worker's understanding. Stay at Phase 2.
- **Divergence on root cause** → `worker-cli send` asking for deeper investigation with specific questions. Read the worker's output first (`worker-cli response`), identify the exact gap between expected and actual, then send the targeted correction. Stay at Phase 2.

**Prohibited:**
- Accepting worker proposals at face value without comparing to your mental model
- "Looks good" rubber-stamping
- Skipping Phase 2 and letting the worker proceed straight to implementation

---

## Worker Phase 3: Go + Implementation

Worker implements after receiving Go. During implementation:

### While Workers Run

**Do NOT poll status repeatedly.** Workers take 2-10 minutes.

**Pattern:**
1. Spawn worker → do independent work (rule edits, other planning, exploration)
2. Timer launch ends the turn → STOP → its completion opens a NEW turn → `worker-cli status` → if idle, proceed with Phase 4
3. Read output with `worker-cli response` after the worker goes idle

### Timer loop

See Core Rules § Timer Loop — After Every Worker Send. Same loop applies here.

### Reading worker output

- While a worker is `working`, only call `worker-cli status` — it's cheap. Don't read output mid-work.
- When idle, read with `worker-cli response`. Use `worker-cli capture` ONLY for a dead / forcefully-stopped worker — it reads the tmux pane directly, since `response` needs a live session.

### File-Move Task Checklist

When a worker task involves moving files to a new subdirectory, the worker must verify EACH of the following AFTER completing the move. Add this checklist explicitly to the worker prompt:

> **File-Move Checklist** — verify EACH point after completing the move:
> 1. **Imports inside moved file:** `.` / `..` prefix depth changed (e.g. `from . import x` → `from .. import x` if now one level deeper). Fix all.
> 2. **Imports outside referencing the moved file:** every caller that imports the old path must be updated.
> 3. **Lazy imports inside functions:** `from . import x` written INSIDE a function body is still a relative import and follows the same rule. Easy to miss — grep explicitly.
> 4. **Grep verification:** `grep -rn 'from \.\|from \.\.' <affected_subdirs> | grep <moved_module_name>` — confirms all references are updated.
> 5. **Smoke test:** run the entry-point or a targeted import check (`python -c "import <top_level_package>"`) to confirm no ModuleNotFoundError.

---

## Worker Phase 4: Review

After worker goes idle, review BEFORE merging.

**A worker's "verified" is a claim, not proof.** Completion Checklists saying "verified" are claims; a worker in a worktree may lack venv or CLI tooling, so its "tests" may never have run. Run the actual test commands yourself — here in review, and again after merge.

### Code Review (MANDATORY)

1. `worker-cli response <name>` → read Completion Checklist
2. Read the worker's complete diff via Bash, NEVER via the Read tool on worktree paths. Canonical command:
   ```bash
   git -C <project_root>/.claude/worktrees/<name> diff integration
   ```
   This shows the full diff from integration tip to the worker's branch tip — every change the worker made, including across multiple commits on the branch. Do NOT restrict to `HEAD~1..HEAD` or `integration..HEAD` (the latter is equivalent but with redundant `HEAD`); code review means reading the entire delta, not only the last commit. For a single file's current content: `git -C <worktree> show HEAD:<relpath>` or `cat <worktree>/<relpath>` via Bash.
3. Check: correctness, existing patterns followed, no regressions
4. If issues found → treat as a review disagreement (see Review Disagreements below)
5. If review passes → proceed to Phase 5

**Non-skippable — even for ad-hoc / one-line / context-recovery merges.** Self-test before EVERY `worker-cli merge`: "Have I run `git -C <worktree> diff integration --` and READ the result in this session?" If no → STOP, run the diff first.

**Sample-Test rendered output (MANDATORY for user-visible features).** When the feature affects formatted output (search results, reports, CLI display, generated text): run ONE live sample and inspect the rendered text — not the parser code that produces it, the actual string the user sees. Code-read does NOT count as sample-test.

**Interpretation Cross-Check (MANDATORY when worker output contains an interpretation of measured data).** Investigation workers often deliver findings narratives that go beyond raw measurements — they interpret the data and propose a mechanism ("data X means mechanism Y"). Before accepting the interpretation:

1. Identify each interpretation claim — a sentence of the form "this measurement means/proves/shows X" or "mechanism is Y".
2. For each claim, locate the source code that produced the data being interpreted. This must be the actual code, in the current src/ tree, read in this session (Step 2 Stage 3 should already have covered it — if not, read it now BEFORE accepting the interpretation).
3. Ask: are there alternative code paths in the same function/module that would produce the same measurement but support a DIFFERENT interpretation? If yes, the worker's interpretation is one of several possible — not proven. Either accept it as one hypothesis among several, or send the worker back with a follow-up probe that discriminates between the candidates.
4. **Reject the interpretation, accept the data.** If the worker's interpretation does not uniquely follow from the source code, the data they collected is still valid evidence — but the conclusion they drew is not yet supported. Phase 5/6 may still proceed (merge the probe artifacts), but the interpretation does NOT become the basis for the next worker's task.

### Review Disagreements

When code review surfaces an issue where YOU and the worker disagree, treat it exactly like a disagreement in § Worker Phase 2: Evaluate — Cross-Model Comparison. Same cross-model gate, no prescribed patch.

**What to check:**
- Does the code address the actual problem (not a symptom)?
- Does it follow existing patterns in the codebase?
- Are there uncommitted local changes that conflict?
- Did the worker commit? (Check Completion Checklist)

**Glue work before merge:** Copy gitignored files, extract configs — anything that lives only in the worktree. Merge deletes the worktree, anything not saved is lost.

---

## Worker Phase 5: Recap (MANDATORY After Every Stage)

**Trigger:** ALWAYS — after Phase 4 Review completes clean for ANY task / etappe, YOU send `worker-cli send <name> "recap"`. Non-discretionary. The recap consolidates DOCS.md sync (code-shape update) and process-docs persistence (write-once entry) into ONE commit while the worker still has the original task context in head.

**Always send recap after a subtask.** Send `recap` after every subtask, no exceptions. YOU just send the trigger; the worker runs its own recap pass, scoped to the subtask it did.

If the worker dies mid-recap, spawn a successor that re-runs the recap for that subtask — see § Worker Death Recovery. NEVER defer drift to session-end RECAP.

**Phase 5 output:** worker commits ONE recap commit (`docs: recap for <task>`), reports touched files + doc updates (DOCS.md / process-docs). Folds into Phase 6 Merge.

---

## Worker Phase 6: Merge + Lifecycle

### Merging

`worker-cli merge <name>` merges the branch into current branch (`integration`). Worker stays alive.

**Pre-Merge Clean-Check (MANDATORY):**
BEFORE `worker-cli merge` / `git merge`: run `git status` in the target repo. If there are uncommitted changes OR untracked files in files that the worker's branch also touches → merge will abort with "your local changes / untracked working tree files would be overwritten". Handle BEFORE merging:
- Uncommitted changes overlapping the worker's diff → `git stash push -u -m "pre-merge <worker>"` first, merge, then decide whether to reapply stash (`git stash pop`) or drop (`git stash drop`). Stale previous-session attempts can safely be dropped AFTER confirming the worker's version contains the same content.
- Untracked files that exactly match what the worker adds → `rm` the file (it's usually a leftover from a previous direct edit), then merge.
- Untracked files unrelated to the merge → leave them alone, merge proceeds cleanly.

**Prevent the conflict at source.** When YOU generate outputs (script runs, smoke reports, measurement files) during a running worker round, write them to `/tmp/` or commit on `integration` immediately (`chore:` commit). Don't leave your own untracked output in paths the worker is editing.

**Post-Merge Verification (MANDATORY):**
- If merge says "Already up to date" → STOP. Worker did NOT commit. Investigate via `worker-cli capture`.
- Run `git diff ORIG_HEAD --name-only` — check expected files are modified. `ORIG_HEAD` = the pre-merge tip, so this shows ALL of the worker's commits, not just the last one (`HEAD~1` would miss earlier commits on a multi-commit branch).
- If no changes: `worker-cli send` with commit instructions

### Worker Lifecycle

**Workers stay alive until they hit their context limit (status → `limit reached`).** We spawn sequentially — one worker at a time — but a second (or Nth) worker may be spawned when the new task touches completely new files and themes, orthogonal to the running worker (§ Reusing Workers). A worker is killed only when it is dead, or at session end when the user explicitly orders the recap.

**When NOT to kill (even if you think it's "done"):**

- After task completion / Phase 6 merge — worker stays idle until the session-end RECAP
- Worker is mid-work (EITHER indicator triggers):
  - Phase A reported but no commit above integration-tip → Phase B blocked on Go, plan lives in worker context
  - `git -C <worktree> status --short` shows uncommitted changes → implementation in flight
- Worker hit a blocker (error/timeout/unexpected state) — `worker-cli send` "Stop, investigate, report" FIRST. Worker has live context (processes, tracebacks, recent reads) that's lost on kill
- Never kill for "low context" — context is not observable. Reuse the worker until it dies (status → `limit reached`), then spawn a successor (§ Worker Death Recovery).

**When TO kill:**

- The worker is dead (status → `limit reached`) — capture its pane first, then kill and spawn a successor (§ Worker Death Recovery)
- Session-end RECAP, on explicit user order
- Worktree filesystem conflict
- Explicit user request

**How to kill:**

`worker-cli kill <name>` — does tmux kill + worktree remove + branch delete in one call.

### Reusing Workers — AGGRESSIVE REUSE (Thematic Continuity)

Reuse the existing worker UNTIL IT DIES. The reuse-vs-fresh decision is about THEMATIC CONTINUITY:

**Reuse the existing worker when:**
- New task touches the same files, packages, or conceptual area as the worker's prior tasks
- New task extends, refines, or builds on committed work the worker did
- Worker has ANY thematic overlap with the new task

**Spawn fresh ONLY when:**
- New task uses files / packages / concepts COMPLETELY ORTHOGONAL to the worker's accumulated context
- Worker is dead (status → `limit reached`) — spawn a fresh successor to continue

**Reuse continues until the worker dies.** A worker keeps receiving follow-ups in its thematic area until its status turns `limit reached`; then spawn a successor (§ Worker Death Recovery).

**Pre-followup Branch Sync (when reusing across merges):** Worker's branch tip is behind current `integration` if merges happened while it was idle. ALWAYS prefix follow-up `worker-cli send` with: "FIRST: in your worktree, run `git -C <worktree-path> fetch origin integration && git -C <worktree-path> merge integration`. Verify with relevant grep. THEN do the work: ..."

### Worker Death Recovery — Successor Continues (Always)

When a worker dies mid-task or mid-recap: ALWAYS spawn a successor. YOU hold the plan — YOU know the task YOU dispatched.

A worker that dies mid-task has committed NOTHING for the in-progress subtask, so commits don't show its progress — the tmux pane does. YOU know what the worker was supposed to do; the pane shows how far it got.

**YOUR role on detected worker death:**
1. `worker-cli capture <name>` FIRST — read how far the dying worker got (last actions, current step). Capture BEFORE killing; kill removes the pane and worktree.
2. Spawn a fresh successor: `worker-cli spawn <successor-name> /tmp/prompt.md <project_path>`
3. Successor prompt = its file complex + the subtask + where to pick up, built from the pane + YOUR plan: "Continue <subtask>; <what's already done per the pane>; resume at <next step>." Completed subtasks are committed on `integration` — the successor builds on that committed state.
4. Phase 2 Cross-Model check on the successor's first response — does it match where the dying worker left off?

**Pre-spawn safety:** if the pane shows the dying worker only read/planned and did nothing yet, the successor gets the ORIGINAL subtask prompt — same as a normal initial spawn.

Kill the dying worker (and remove its worktree) only AFTER capturing its pane and the successor has started. `worker-cli kill` is irreversible — capture first.

### After Deliverables Complete

**1. Report deliverable status in chat — prose, no table.** For each deliverable, a short paragraph: what was done, its status (done / partial), and how YOU verified it (code review / test run / not verified). Be brutally honest — code read ≠ verified.

**Code Review happens on `integration` branch** (normal project path), NOT by reading worktree files.

**2. Scope user verification (STOP)**

For each deliverable: propose a concrete verification step the **user** can perform as the final quality gate.
- What exactly to click, run, or check
- What the expected output or behavior is

Wait for remarks. When user has no remarks → run verification together.

---

## Session Recap

Decoupled from the worker cycle — runs at the very END of the session, on explicit user trigger ("wir machen session recap"). It is NOT a phase of any PLAN/IMPLEMENT loop; the user decides when it happens.

**Scope (concern separation):** YOUR Session Recap covers ONLY:
1. **Files YOU touched directly** — rule files in `~/.claude/`, issues, RAG sync, cross-project edits
2. **Worker omissions YOU noticed** — drift YOU spotted during Phase 4 Review or post-merge verification that the worker missed. Document the omission as a session-end fix.

That's it. Workers do worker recaps for their tasks — fully self-contained. YOU do YOUR recap for YOUR surface. **YOU NEVER recap a worker's task surface** — if a worker recap was incomplete, the fix is to either (a) catch it in Phase 4 Review and dispatch a follow-up worker, or (b) note the omission in YOUR session-end recap WITHOUT redoing the worker's job.

YOU NEVER deliberately move drift to session-end. If a session-end RECAP finds substantial drift from a completed worker task that the worker should have covered, that's a process violation — investigate why Phase 4 Review didn't catch it and adjust next session.

Two phases. ONE stop between them.

- `🔍 RECAP` — issues + doc-file plan → short Chat summary → STOP for remarks
- `🛠️ IMPROVE+CLOSE` — execute, no further stops

### 🔍 RECAP

#### Issues Evaluation

`gh-cli list_issues <owner> <repo>`. For each open issue decide: CLOSE / UPDATE Source-Inventory / CREATE.

Source-Inventory updates live in the issue BODY (read via `get_issue`, splice in new source paths, full-replace via `update_issue --body`) — there are no comments. Narrative goes to process-docs / DOCS.

**EMPTY PLATE:** every Open Item from the original plan not executed → capture it before closing. Usually a process-docs entry; an Issue only when it's a standalone task in its own right.

#### Chat summary

- **Issues:** which issues we touched this session, which get newly created, which get closed, which get a Source-Inventory update.
- **Doc files:** which doc files (process-docs / DOCS) get written or edited in the IMPROVE phase, and which doc-file paths get added to or removed from which issue's Source-Inventory.

🛑 STOP — ask "Bemerkungen?"

### 🛠️ IMPROVE+CLOSE

One run through, no stops.

1. **Execute the Chat summary** — write the process-docs / DOCS files and do the issue hygiene (create / close / Source-Inventory update) exactly as named in #### Chat summary above.
2. **Sync docs to RAG** — `[ -f .rag-docs.json ] && rag-cli update_docs .` (skipped silently when no manifest). RAG sync runs ONLY here at recap — NEVER mid-session.
3. **Git closing** — `git checkout main && git merge integration` → per repo: `git-check` → commit → push (or `plugin-publish` for plugin repos).

Done when commits are pushed.
