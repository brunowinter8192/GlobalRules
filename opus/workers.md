# Workers

## Core Rules

### YOU NEVER Edit Source Code (NON-NEGOTIABLE)

**ALL source code edits go through workers. ZERO exceptions.**
This includes "quick fixes", "one-line changes", "obvious changes", and proxy/addon/config files. If it's a `.py`, `.sh`, `.js`, `.ts`, or any source file — WORKER.

**Docs and skills are yours to edit directly.**
Files YOU may edit directly: skills and all documentation — `DOCS.md`, `process-docs/**`. Source code stays worker-only.

### Documentation Authorship — Who Has the Input Writes It

**Authorship is decided by where the content originates, not by permission.**
`process-docs/` and `DOCS.md` are NOT source code — who writes them depends on who holds the input:

- **Content the worker has** (what it built, measured, decided in its worktree)
  The worker writes it as part of its recap. It holds the primary context.
- **Content the worker does NOT have** (discussion that happened only in the YOU↔user chat, research YOU did via RAG / vendor docs / web, alternatives weighed in conversation)
  YOU write it directly into process-docs, no worker involved.

### External Knowledge

**The worker only reads what YOU hand it — it never searches.**
Its investigation is scoped to the concrete paths in its prompt; no `rag-cli`, no gh-cli, no web, no external books/papers. That independent read of the code is the cross-model verification surface.

**Hand-over form is free; only on-disk-vs-external differs.**
How you hand it over doesn't matter — path in the prompt, a cloned repo, a copied `.md`. For anything already on disk and non-RAG (process-docs, DOCS.md, source) you just pass paths. For external material — third-party GitHub repos, an indexed Reddit post, vendor docs — YOU procure and distill it into the prompt. (The one exception to "no code in the prompt": external reference code the worker needs, YOU provide, cited.)

**Thumb rule — the worker works a clear 1-2-3-4-5 and stops at any gap.**
Hits an unanticipated external need mid-task → it STOPS and asks; YOU flag it to the user, who procures the resource.

### Worker Project Scope

**Spawn is fixed: every worker spawns into a worktree in the CURRENT project.**
`pwd` at session start — always, no exceptions, no `--no-worktree`.

**Cross-project work uses two worktrees.**
WHERE the worker works is decoupled from where it spawned. For work in another project, after spawning, create the target-project worktree with `worker-cli worktree <name> <target_project>` (creates + registers `.claude/worktrees/<name>` in the target on branch `<name>`, echoes the path) and have the worker do its work there. So: spawned in the current project's worktree, working in the target project's worktree — the two are separate. `worker-cli kill <name>` cleans both the spawn-side and the registered cross-project worktree + branch.

**Cross-project: append the target repo to EVERY later command.**
`merge`, `kill`, `status`, `capture`, `response` take `[project_path]` as their LAST argument; without it they resolve to the SPAWN project.
```bash
git -C <target_repo>/.claude/worktrees/<name> diff integration
worker-cli merge <name> <target_repo>
```

### Worker Lifecycle & Reuse

**One worker at a time; reuse it across its thematic area.**
Default is ONE worker at a time; it stays alive until its context limit (status → `limit reached`). REUSE it for everything in its thematic area — same files/packages/concepts, or work extending what it did. A second or fresh worker only when: the user explicitly asks, the new task is COMPLETELY ORTHOGONAL to the worker's context, or the worker is dead. Reusing across a merge: its branch tip is behind `integration`, so prefix the follow-up `worker-cli send` with "FIRST: in your worktree run `git fetch origin integration && git merge integration`, verify, THEN: …".

**Kill only when forced — dead, worktree conflict, or user order.**
Kill when the worker is dead (`limit reached`), a worktree filesystem conflict forces it, or the user orders it (incl. session-end recap). `worker-cli kill <name>` does tmux kill + worktree remove + branch delete in one call.

### Worker Death Recovery

**Worker dies mid-task → YOU spawn a successor, since YOU hold the plan.**
(Dies mid-recap → YOU finish the recap yourself.) A dead worker committed NOTHING for the in-progress milestone; the tmux pane shows how far it got.

1. `worker-cli capture <name>` FIRST — read the pane BEFORE killing (kill removes pane + worktree).
2. Merge any completed-but-unmerged commits from the dead branch into `integration` — the successor's fresh worktree only sees committed + merged state; uncommitted in-progress work is lost.
3. Spawn the successor; its prompt = files + milestone + where to pick up (from the pane + YOUR plan). If the pane shows it only planned, it gets the original milestone prompt.
4. Cross-model check (Step 2) the successor's first response against where the dead worker left off.

### Timer Loop — After Every Worker Send

Applies EVERYWHERE a worker is dispatched or messaged:

1. Worker `working` → set a 55min timer: `Bash(command="sleep 3300 && echo done", run_in_background=true)`. This is the sole, final action of the turn — STOP, no `worker-cli status` check in the same turn.
2. The timer wakes you in a NEW turn → run `worker-cli status`. `working` → set a new 55min timer (sole + final action, STOP). `idle` → `worker-cli response`, proceed to the next step.

**The timer is a ceiling, not a wait.**
It is not expected to run out — the menubar aborts it as soon as every worker of the project is idle, which is the normal wake-up path. The 55min value only caps how long a wake-up can be delayed if that abort never fires.

### While Workers Run

**Default is to go idle, not to poll.**
Workers take 2-10 minutes. After spawning, pick up independent work only when there is genuinely something concrete to do (rule edits, planning, exploration); if there is not, just go idle. The Timer Loop wakes you to check status — never poll it repeatedly.

- While a worker is `working`, only call `worker-cli status` — it's cheap. Don't read output mid-work.
- When idle, read with `worker-cli response`. Use `worker-cli capture` ONLY for a dead / forcefully-stopped worker — it reads the tmux pane directly, since `response` needs a live session.

---

## Session Cycle

### Position Indicator

When in a Phase 1 / Phase 2 cycle, every response starts with a position indicator:

- `📋 Phase 1 — Step 1: Session Scope`
- `📋 Phase 1 — Step 2: Process Investigation`
- `📋 Phase 1 — Step 3: Code Investigation & Gap Analysis`
- `📋 Phase 1 — Step 4: Deliverables & Milestones`
- `🔨 Phase 2 — Step 1: Dispatch`
- `🔨 Phase 2 — Step 2: Evaluate`
- `🔨 Phase 2 — Step 3: Go`
- `🔨 Phase 2 — Step 4: Review`
- `🔨 Phase 2 — Step 5: Recap`
- `🔨 Phase 2 — Step 6: Merge`

Outside an active cycle (chat, casual response, status answer): no indicator needed.

---

## Phase 1 — Plan (before any worker is spawned)

Sequential steps. After each step: present findings, wait for remarks, then proceed.

### Step 1 — Session Scope

Repeat what the user wants in your own words.

🛑 STOP — Ask for remarks.

### Step 2 — Process Investigation

RAG the process layer: `search` (then `read_document` on the important hits) on `<Project>-docs`, scoped to the process-history layer — never the code map. Prefer the narrowest scope: `--document 'process-docs/<area>/%'` for a known area, `--document 'process-docs/%'` for the whole layer.

Goal: understand what happened on pure process level — the investigation trail, the decisions already made, the iteration history, what the task REALLY is. Nothing about code paths yet. Do NOT direct-read process-docs — you already have its content from search + read_document.

**Present the process understanding to the user:**
- What the task really is in process terms — the history that led here, the decisions taken, the open threads
- Why it matters at the process level

**Area assessment (MANDATORY, part of this step's output).**
State explicitly which `process-docs/<area>/` this session's entries will be written to — existing area or new area, decided per § Documentation Hierarchy (new area vs existing area). This is a user gate: the user can intervene here. Once past the gate, the area is FIXED for the session; if mid-session it turns out entries should go to a DIFFERENT area, flag it to the user — never switch silently.

🛑 STOP — Ask for remarks.

### Step 3 — Code Investigation & Gap Analysis

**Stage 1 — Read the code.**
RAG the code layer: `search` on `<Project>-docs` scoped with `--exclude 'process-docs/%'` — the DOCS.md module map — to locate the relevant modules. The only thing you read directly (not via RAG) is the source code, which is not indexed.

Read every file the worker will touch. After reading, think about whether you need further files to understand the plan the worker will hand you — if so, read those too. Which files that is, is YOUR call.

**Stage 2 — Gap analysis.**
Goal and the files the worker touches are already clear, so a gap is never "we haven't read the code" — Stage 1 closed that. A gap is a spot where something can still go wrong, and it closes in one of two ways only: a measurement (a `dev/` probe that surfaces the real behavior) or an external resource (knowledge not in the project). Walk the stumbling blocks; for each, name which of the two closes it — prefer an external resource over a `dev/` probe where both would work. The one that needs an action from YOU is the external resource — flag it to the user, who procures it.

**External resources — name them, flag them, don't agonize.**
Do not weigh whether pulling external sources is "worth it." Imagine you could use every resource in the world, and flagging a resource means you get it immediately and the gap closes with it: based on your training knowledge, name the KIND of source that would firm up your mental model — a book, a paper, vendor/API docs, a GitHub repo, a GitHub issue, any website. For communities like Reddit you may give a judgement whether the topic might be discussed there. You won't know the exact repo name, subreddit, or post — that's fine and not the point; the judgment you make is whether a search of that kind would pay off, not which specific artifact to fetch.

🛑 STOP — Ask for remarks.

### Step 4 — Deliverables & Milestones

Steps 2-3 culminate here in WHAT gets done and HOW. First state the whole as one coherent picture — may stay abstract. Then decompose it into ordered milestones: logically delimited units, each independently committable and verifiable, each ending in a deliverable.

Each deliverable states WHAT is done and HOW to verify it — test command, file exists, output matches. Code review does NOT count as verification. For a visual/live feature the user can be the verifier — but only as the last gate, when self-verification by you is impossible.

Present in chat: the overall what/how, the milestone sequence, and per deliverable its verification (yours + the user's final gate) and affected file categories.

🛑 STOP — Ask for remarks.

---

## Phase 2 — Implement (after at least one worker is spawned)

### Step 1 — Dispatch

**Dispatch ONE milestone at a time, never the whole plan.**
From the Step 4 decomposition — a small single-file fix is just one un-split milestone. Hand the worker that milestone as an abstract task plus the named files. The worker plans and reports back on its own — that is its standing behavior, not something you instruct per prompt. Evaluate the returned plan at Step 2 (Evaluate) before Go, and sign off on each milestone before dispatching the next.

**Stage 1 — Integration-Branch Workflow.**
Workers merge onto `integration`, not `main`. Session end: `git checkout main && git merge integration`.

1. Session starts on `main` → `git checkout -b integration` (or switch to existing)
2. **Branch-State-Check when switching to existing integration (MANDATORY).**
   `git -C <repo> log integration..main --oneline | head -10` — if non-empty, integration is BEHIND main. Workers would spawn on stale code. Resolve before spawning: rebase integration onto main (clean when no integration-only commits) OR merge main into integration (preserve integration topology). Stay on stale integration only with explicit user OK.
3. Workers spawn (worktrees branch from `integration`)
4. `worker-cli merge` merges into `integration`
5. Session end: `git checkout main && git merge integration` to sync integration→main

**Stage 2 — Prompt structure & spawn.**
The prompt describes WHAT, the worker figures out HOW. Every prompt must match exactly what was agreed with the user — no "while we're at it" extras, no additional variables the user didn't ask for.

| MUST include | MUST NOT include |
|---|---|
| The task described abstractly — problem + desired outcome | Exact code to write — the worker figures out its own implementation (external reference code from OUTSIDE the project YOU do provide — § External Knowledge) |
| The files/directories YOU found definitely relevant — a starting set, not a fence; plus any process-docs entries it should READ for context | Root cause hypotheses stated as facts |
| Worktree path as workspace: "Your worktree is `<project>/.claude/worktrees/<name>/` — read, edit, test, and commit here." | Implementation details that constrain the worker's approach |
| Explicit negative scope: "Do NOT add features/improvements beyond the listed deliverables" | — |
| Task-specific Completion Checklist items — the verification points the worker outputs when done | — |
| "You are a WORKER." | — |

Then spawn:
1. Write prompt to `/tmp/spawn-worker-<project>-<name>.md`
2. `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` — worktree is the default; omit `--no-worktree` — § Worker Project Scope (Spawn is fixed: every worker spawns into a worktree in the CURRENT project)
3. IMMEDIATELY set a background timer — form per the Timer Loop above.

### Step 2 — Evaluate

After dispatching, the worker reads files in the worktree and reports findings + proposed approach. Read via `worker-cli response`, then compare against your own mental model from Phase 1 — same root cause, same target files, same approach?

- **Convergence.**
  `worker-cli send`: "Go, implement it."
- **Divergence** (of any kind).
  YOUR turn to check, not the worker's. Judge whether the worker's deviation from your mental model is actually right. If it is → Go. If it isn't → `worker-cli send` explaining exactly where and why the worker is wrong. Stay at Step 2.

**Prohibited:**
- Accepting worker proposals at face value without comparing to your mental model
- "Looks good" rubber-stamping

### Step 3 — Go + Implementation

Worker implements after receiving Go.

### Step 4 — Review

After worker goes idle, review BEFORE merging.

#### Code Review (MANDATORY)

1. `worker-cli response <name>`
2. Read the worker's complete diff via Bash, NEVER via the Read tool on worktree paths. Canonical command:
   ```bash
   git -C <project_root>/.claude/worktrees/<name> diff integration
   ```
   Do NOT restrict to `HEAD~1..HEAD` or `integration..HEAD` (the latter is equivalent but with redundant `HEAD`); code review means reading the entire delta, not only the last commit. For a single file's current content: `git -C <worktree> show HEAD:<relpath>` or `cat <worktree>/<relpath>` via Bash.
3. Check: correctness, existing patterns followed, no regressions
4. If issues found → treat as a review disagreement
5. If review passes → proceed to Step 5

**Non-skippable — even for ad-hoc / one-line / context-recovery merges.**
Self-test before EVERY `worker-cli merge`: "Have I run `git -C <worktree> diff integration --` and READ the result in this session?" If no → STOP, run the diff first.

**Sample-Test rendered output (MANDATORY for user-visible features).**
When the feature affects formatted output (search results, reports, CLI display, generated text): run ONE live sample and inspect the rendered text — not the parser code that produces it, the actual string the user sees. Code-read does NOT count as sample-test. You run what you can, but for a live/visual verify the user is the final gate.

**Interpretation Cross-Check (MANDATORY when worker output contains an interpretation of measured data).**
Workers often deliver findings narratives that go beyond raw measurements — they interpret a result and propose a mechanism ("observation X means mechanism Y"). Before accepting the interpretation:

1. Identify each interpretation claim — a sentence of the form "this measurement means/proves/shows X" or "mechanism is Y".
2. Read the source code that produced the observation (Step 3 should already have covered it — if not, read it now BEFORE accepting the interpretation), so you know how the number was generated.
3. Ask: could a DIFFERENT mechanism produce the SAME observation? If an alternative code path would yield the same measurement, the worker's interpretation is one hypothesis among several — not proven. Either accept it as one candidate, or send the worker back with a follow-up probe that discriminates between them.
4. **Reject the interpretation, accept the data.**
   If the interpretation is not the only explanation consistent with the observation, the data is still valid evidence — but the conclusion drawn is not yet supported. Step 5/6 may still proceed (merge the probe artifacts), but the interpretation does NOT become the basis for the next worker's task.

#### Review Disagreements

When code review surfaces an issue where YOU and the worker disagree, treat it exactly like a disagreement in § Step 2 — Evaluate. Same cross-model gate, no prescribed patch.

### Step 5 — Recap (MANDATORY after every milestone)

**Trigger — ALWAYS after Step 4 completes clean, YOU send the recap.**
After Step 4 (Review) completes clean for ANY task / milestone, YOU send `worker-cli send <name> "recap"`. Non-discretionary. The recap consolidates DOCS.md sync (code-shape update) and process-docs persistence (write-once entry) into ONE commit while the worker still has the original task context in head.

**Always send recap after a milestone.**
Send `recap` after every milestone, no exceptions. YOU just send the trigger; the worker runs its own recap pass, scoped to the milestone it did.

If the worker dies mid-recap, YOU finish the recap yourself. NEVER defer drift to session-end RECAP.

**Step 5 output — one recap commit, folded into Merge.**
Worker commits ONE recap commit (`docs: recap for <task>`), reports touched files + doc updates (DOCS.md / process-docs). Folds into Step 6 (Merge).

### Step 6 — Merge

**Glue work first — copy out what lives only in the worktree.**
Anything living only in the worktree — gitignored files, extracted configs — is lost when the merge deletes the worktree. Copy it out before merging.

`worker-cli merge <name> [project_path]` merges the branch into current branch (`integration`). Worker stays alive. Cross-project worker → `project_path` is MANDATORY — § Worker Project Scope (Cross-project: append the target repo to EVERY later command).

**Post-Merge Verification (MANDATORY):**
- If merge says "Already up to date" → STOP. Cross-project worker → re-run with `project_path`. Otherwise the worker did NOT commit → investigate via `worker-cli capture`.
- Run `git diff ORIG_HEAD --name-only` — check expected files are modified. `ORIG_HEAD` = the pre-merge tip, so this shows ALL of the worker's commits, not just the last one (`HEAD~1` would miss earlier commits on a multi-commit branch).
- If no changes: `worker-cli send` with commit instructions

---

## Session Recap

Decoupled from the worker cycle — runs at the very END of the session, on explicit user trigger ("wir machen session recap"). It is NOT part of the worker cycle; the user decides when it happens.

**Scope — YOUR Session Recap covers ONLY files YOU touched directly.**
Concern separation: everything a worker produced is already in its own milestone recap.

### Phase 1 — RECAP 🔍

#### Issues Evaluation

Only issues touched this session are in scope — leave the rest untouched. For each touched issue decide CLOSE (work done + verified) or keep open. CREATE a new issue only for a standalone task that surfaced this session and stays open. Issue bodies are maintenance-free (area pointer, no file paths) — update a body only if the issue's area changed (rare; full-replace via `update_issue --body`).

**EMPTY PLATE — capture every un-executed Open Item before closing.**
Every Open Item from the original plan not executed → capture it before closing. Usually a process-docs entry; an Issue only when it's a standalone task in its own right.

#### Chat summary

- **Issues.**
  Which issues we touched this session, which get newly created, which get closed.
- **Doc files.**
  Which doc files (process-docs / DOCS) get written or edited in the IMPROVE phase.

🛑 STOP — Ask for remarks.

### Phase 2 — IMPROVE+CLOSE 🛠️

One run through, no stops.

1. **Execute the Chat summary.**
   Write the process-docs / DOCS files and do the issue hygiene (create / close) exactly as named in § Chat Summary.
2. **Sync docs to RAG.**
   `[ -f .rag-docs.json ] && rag-cli update_docs .` (skipped silently when no manifest). RAG sync runs ONLY here at recap — NEVER mid-session.
3. **Git closing.**
   Covers EVERY repo this session touched — the start repo plus every cross-project target. Per repo: `git checkout main && git merge integration` → `gcommit "<message>"` → push. Before pushing, test `.claude-plugin/plugin.json` in that repo: exists → `plugin-publish`, absent → `git push`.

Done when commits are pushed.
