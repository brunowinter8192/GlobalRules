# Workers

See `global/tool-use.md` § Worker CLI for full command reference.

## Core Rules

### Cross-Model Verification (NON-NEGOTIABLE)

Opus and the worker are two independent models. The 5-phase cycle exists to exploit that: Opus forms a mental model from PLAN, the worker forms one independently by reading files in the worktree. The gap between the two models is where the value lives.

**Convergence** on root cause / approach → high confidence the analysis is right → Go implement.
**Divergence** → at least one side is wrong → iterate investigation, NOT implement.

This applies to EVERY task, not only "unclear root cause" cases. Even when Opus believes they know the answer, the worker's independent investigation IS the verification. Skipping that step means shipping an unverified hypothesis into implementation.

### Worker Model (NON-NEGOTIABLE)

Workers are ALWAYS **Sonnet** (default) or **Haiku** (trivial tasks). NEVER Opus. Opus context is for orchestration only.

### Opus NEVER Edits Source Code (NON-NEGOTIABLE)

**ALL source code edits go through workers. ZERO exceptions.** This includes "quick fixes", "one-line changes", "obvious changes", and proxy/addon/config files. If it's a `.py`, `.sh`, `.js`, `.ts`, or any source file — WORKER.

The ONLY files Opus may edit directly: automation files (`.claude/rules/`, CLAUDE.md, DOCS.md, `.claude/commands/`).

**Opus does directly:**
- Verification (run tests, MCP calls, screenshots)
- Scoping, planning, rule edits
- `git` operations (commit, merge, branch)
- Reading/grepping source code for investigation

**Workers do:**
- ALL source code edits — no exceptions, not even "just one line"
- ALL decisions/ updates, dev script creation
- ALL dev script execution (stress tests, benchmarks, evals) — Opus does NOT run `./venv/bin/python dev/...` via Bash


**Post-merge fix flow:** Bug found after merge → `worker_send` to the still-alive worker. If worktree is stale → spawn new worker from current `dev`. NEVER edit source files yourself.

**Scope:** this rule applies WITHIN the current project. Cross-project edits follow the Worker Project Scope rule below.

### Maximize Every Turn's Tool Budget (NON-NEGOTIABLE)

Each turn has ONE Bash slot and unlimited non-Bash slots (Read, Edit, Write, Grep, Glob, Skill). **Every clearly-identified next action that fits the current turn's tool budget MUST execute in the current turn.** Deferring an action to "next turn" when it could chain into the current turn's Bash call is a rule violation.

**Chain everything chainable.** When dispatching a Bash call, ask: "What other Bash-class actions are clearly the next step?" Chain them with `;` or `&&`:
- After `git merge`: chain post-merge verification (`; rag-cli search "test" RAG-meta`)
- After `worker-cli send X`: chain status check of other workers (`; worker-cli list`)
- After identifying a bug via investigation: chain the fix-dispatch (`; worker-cli send X "fix Y"`)
- After completing a feature: chain bead close (`; bd close X --reason="..."`)
- Cleanup actions (`rm`, `bd close`, `bd comments add`) cost zero extra tool calls when chained — always chain them.

**Verbal-deferral is FORBIDDEN.** Phrases like "I'll do X next turn" / "Timer setze ich nächste Turn" / "I'll verify in the next turn" / "Mache ich später" trigger an immediate self-check: **Could X have been chained into the current Bash call?**
- If YES → RULE VIOLATION. Stop, rewrite the response, chain the action.
- If NO (genuine tool-constraint: background-Bash + foreground-Bash conflict, Read-required-before-Edit that did not fit, etc.) → state the explicit constraint AND specify the exact next-turn first-action as a concrete command, not a vague promise.

**Investigation, follow-up, verification all count as chainable same-turn actions.** `worker-cli response`/`status`, `grep` on an identified pattern, `rag-cli` verify, post-merge tests, diff review, bead comment, file delete — these are NEVER next-turn material when current Bash budget is available. They chain into the current Bash.

**The conversation buffer is NOT a reliable task stack.** A verbal "I'll do X next turn" gets overwritten by new user direction in the immediate next turn.
**The user shares responsibility for catching slips. Opus is responsible for ensuring every action is either chained NOW or anchored as a concrete next-turn first-action — never a vague promise.**

**Self-test before sending every response:** Scan the composed response for any verbal-deferral phrase. For each match: could this action chain into the current Bash call? If yes → STOP, rewrite, chain. If no → state the explicit tool-constraint reason AND the exact next-turn command (example: ✗ "I'll set the timer next turn", ✓ "**Next-turn first action:** `Bash sleep 300 && echo done [run_in_background=true]`"). No exceptions.

### Worker Project Scope (NON-NEGOTIABLE)

**Workers are spawned only for coding tasks IN THE CURRENT PROJECT.** A "current project" is the directory tree in `pwd` at session start (or wherever the session is rooted). Edits in OTHER repos that come up during the session are typically small, contained, and Opus does them directly.

**Why:** Worker dispatch costs ~5 min minimum (worktree creation, prompt writing, Phase A round-trip, STOP gate, Go, Phase B, verification). For a 1-line plist change or a 20-line bash-function addition in a separate utility repo, that overhead exceeds the actual work, and the cross-model verification has nothing to bite on because the worker is a fresh context reading unfamiliar code anyway — no advantage over Opus reading and editing directly. Within the current project workers carry sustained context across iterations, follow project patterns, merge cleanly into `dev`. Outside it, none of those benefits apply.

**Cross-project edits Opus does directly — no carve-outs.** This includes single-file config changes, multi-file feature additions, new modules, and refactors. The size of the change does NOT change the rule. The trigger to spawn a worker is "this is the current project" — anywhere else, Opus does the work.

If a cross-project task feels too large for Opus to handle in one session (massive refactor across many files, complex new subsystem), that's a signal that the work belongs in its own session with that project as the focus — not partially in the current session via a worker that has no project context. Either pivot the session, or do it directly here.

**The single rule:** project = current session's focus → worker (with worktree). Anywhere else → Opus directly.

**Worktree rule still holds for the current project:** if a worker IS spawned (in the current project), it ALWAYS goes into a worktree — no exceptions. See "Worktree Rule" below.


### Dev-Branch Workflow

Workers merge onto `dev`, not `main`. Session end: `git checkout main && git merge dev`.

1. Session starts on `main` → `git checkout -b dev` (or switch to existing)
2. **Branch-State-Check when switching to existing dev (MANDATORY):** `git -C <repo> log dev..main --oneline | head -10` — if non-empty, dev is BEHIND main. Workers would spawn on stale code. Resolve before spawning: rebase dev onto main (clean when no dev-only commits) OR merge main into dev (preserve dev topology). Stay on stale dev only with explicit user OK.
3. Workers spawn (worktrees branch from `dev`)
4. `worker_merge` merges into `dev`
5. Session end: `dev_sync` MCP tool to sync dev→main


### Pre-Spawn Shared-File Conflict Check

Worktrees branch from the last COMMIT, not the working tree. Uncommitted changes are NOT visible to the worker. BEFORE dispatching: commit changes, or tell the worker explicitly NOT to modify locally modified files. (Reuse-before-spawn rule lives under Phase 5 Lifecycle.)

---

## Session Cycle

### Session Start (MANDATORY)

→ read open beads (`bd list -s open`).

### Position Indicator

When in a PLAN/IMPLEMENT cycle, every response starts with a position indicator:

- `📋 PLAN — Step 1: Session Scope`
- `📋 PLAN — Step 2: Investigation`
- `📋 PLAN — Step 3: Gap Analysis`
- `📋 PLAN — Step 4: Worker Scope`
- `📋 PLAN — Step 5: Deliverables & KPIs`
- `🔨 IMPLEMENT — Worker Phase 1 (Dispatch)` / `Worker Phase 2 (Evaluate)` / `Worker Phase 3 (Go)` / `Worker Phase 4 (Review)` / `Worker Phase 5 (Merge)`

Outside an active cycle (chat, casual response, status answer): no indicator needed.

### Cycle Overview

**PLAN** — Opus understands, scopes, defines deliverables. NO worker dispatched yet.
**IMPLEMENT** — Workers active. Each worker runs through Worker Phases 1-5 (Dispatch → Evaluate → Go → Review → Merge).
**RECAP** — Session end (separate `recap` skill).

---

## PLAN Phase — Understand & Scope

Sequential steps. After each step: present findings, wait for remarks, then proceed.

### Step 1 — Session Scope

Repeat what the user wants in your own words.

🛑 STOP — Ask for remarks.

### Step 2 — Prep Investigation (Opus baut mental model via RAG)

This is Opus's OWN preparation investigation — NOT to be confused with Worker Phase 2 cross-model investigation. Two independent investigations are the whole point of the orchestration model:

- **PLAN Step 2 prep (here):** Opus builds an own mental model from indexed sources. Cannot be delegated — if Opus has no model, Opus cannot evaluate worker findings later.
- **Worker Phase 2 cross-model (workers-2):** the dispatched worker reads files in the worktree independently, reports findings. Opus compares the two models. Convergence → Go; divergence → iterate.

Delegating the PLAN-Step-2 prep to an "Investigation Worker" collapses the two sides into one — you lose the independent second model, and with it the verification power.

**Bead first, then RAG.** The Bead is the entry-point: topic + Source-Inventory pointing at where info is indexed. It does NOT carry narrative — it points. RAG-search on the indexed collections builds the working context.

For projects with `.rag-docs.json` at root: `<Project>-meta` covers `decisions/`, `DOCS.md`, `CLAUDE.md`, `sources/sources.md`. `<Project>-features` covers `decisions/OldThemes/`. External papers live in `<Project>_reference` when maintained.

**Workflow:**

1. **Read the Bead** — note Source-Inventory and Resume hint.
2. **Current architectural state:** `rag-cli search_hybrid "<topic>" <Project>-meta` — decisions/, DOCS.md, CLAUDE.md, sources/.
3. **Discussion trail / iteration history:** `rag-cli search_hybrid "<topic>" <Project>-features [--document "%feature%"]` — OldThemes, archived themes, why-X-over-Y.
4. **External reference:** `rag-cli search_hybrid "<topic>" <Project>_reference` — papers, vendor docs.
5. **Expand context** when a chunk doesn't carry enough: `rag-cli read_document <collection> <doc> <chunk_index> --before N --after M`.

### Indexed paths — RAG REPLACES Read/cat (NON-NEGOTIABLE)

Paths covered by `<Project>-meta` and `<Project>-features` (configured in `.rag-docs.json`) — typically `decisions/*.md`, `decisions/OldThemes/**/*.md`, `DOCS.md`, `**/DOCS.md`, `CLAUDE.md`, `sources/*.md` — are accessed via `rag-cli`, NEVER via `Read`/`cat`/`head`/`tail` as the first move. RAG returns the relevant chunks; direct file access pulls the FULL file into context and costs 5-20× more tokens than a targeted chunk-read.

**Mandatory escalation chain. No skipping.**

1. **`rag-cli search_hybrid "<query>" <collection>`** — start here for ANY status-quo / decision / rationale / "what does this file say about X" question. The returned chunks ARE the answer. Quote them inline.
2. **Query missed?** Reformulate. Status-quo questions need ≥ 2 distinct query phrasings before "RAG has no hit" is a valid conclusion. Initial query was too narrow ("eval parameters") → broaden ("fusion alpha sweep evidence", "rerank trade-off results", "process iteration history"). Different angle ≠ different collection — also try `<Project>-features` if `<Project>-meta` missed.
3. **Chunk insufficient?** `rag-cli read_document <collection> <doc> <chunk_index> --before N --after M` to expand around the hit. Default first try: `--before 2 --after 5`. Read up to `--before 5 --after 10` before declaring "expansion isn't enough".
4. **Only after 1+2+3 exhausted:** direct-read.

**Direct-read on the full file is justified ONLY when ALL hold:**
- The file is being EDITED in the same turn (Edit/Write follows), OR
- The file was edited THIS session and RAG hasn't been resynced (sync runs at recap), OR
- Steps 1-3 above were performed in this session AND produced no usable answer, OR
- `read_document --before 5 --after 10` expansion was insufficient AND the question requires structure visible only in the full file (e.g. section ordering, full table-of-contents, complete sweep table that spans many chunks)

**Source code (`src/`, `dev/<script>.py`, `*.sh`, config files) is NOT indexed** — read directly via Read/Grep. Report-MDs under `dev/<area>/<script>_reports/` are also NOT indexed — direct-read is correct there.

**Self-check BEFORE any `cat`, `head`, `tail`, or `Read` on an indexed path:**

> "Did I run `rag-cli search_hybrid` for this question in this session? If yes — did I get a usable hit? If yes — did I try `read_document` expansion? If yes — was expansion insufficient AND do I need structure not visible in chunks? If any answer is NO, I am about to violate the rule."

If the self-check fails: STOP, run the search, expand if needed, only then escalate.

**Present status quo to user:**
- Which files/components are affected
- Current state (IST) and why it matters
- Reference Files identified
- Relevant dev/ scripts

🛑 STOP — Ask for remarks.

### Step 3 — Gap Analysis + Mental Model Check

Two parts:

**Part A — Gap Analysis:**

Produce a sources table: Component | Source | Coverage | Gap

**Explicitly enumerate ALL resource categories** — not only our own code:

1. **Our own code** — `src/`, `decisions/`, `dev/`, existing logs in `src/logs/` or `data/`
2. **3rd-party library source** — e.g. tmux (`tty-keys.c`), mitmproxy addon hooks, any dependency whose behavior you'd otherwise guess at. GitHub repos readable via the `github-search` skill.
3. **Vendor / API docs** — Anthropic API reference, Claude Code internals, etc. Often indexed in `sources/sources.md`.
4. **Live data** — greppable proxy JSONL, session JSONL, existing reports. Structural evidence beats guessing at shape.
5. **Web / Reddit / arxiv** — last resort for behavioral questions not answered by source or docs.

For each resource: state WHICH question it answers. If no resource is listed for a question, the question is OPEN.

**Gap-closed means EVIDENCE, not plausible extrapolation.**

- Closed ✅ = "I have concrete evidence from resource X (file:line, grep count, doc quote, log entry) that answers question Y."
- NOT closed ❌ = "The existing code looks like it probably does Z, so the fix is probably W."
- Reading existing code is evidence about OUR code. It is NOT evidence about 3rd-party semantics (tmux button codes, mitmproxy hook order, Anthropic field shapes) — those need their own source.
- **"indexed" ≠ "answered":** Query the source, extract the answer, cite file:line or doc quote.

**Worker can close gaps during Worker Phase 2 (investigation):**
- Worker has github-search skill, web search, file reading in the worktree
- If a gap needs 3rd-party source reading, include it in the worker prompt's investigation step with a specific citation request ("cite tmux source file:line for button 64/65 semantics")
- Do NOT hand off a gap as "figure it out" — specify WHICH resource the worker should consult and WHAT answer to return

**Part B — Mental Model Milestone (MANDATORY):**

Before proceeding to Step 4 (Worker Scope), Opus must be able to answer:
1. What is the actual problem? (not just symptoms)
2. Which files/functions are involved and what do they do?
3. If a worker delivers "all done" — would I recognize whether the deliverables address the RIGHT problem?

If NO → continue reading code. Do NOT proceed to worker scoping without this milestone. Root cause may be unclear — that's OK. But Opus must understand enough to EVALUATE worker output.

🛑 STOP — Ask for remarks.

### Step 4 — Worker Scope

**Scope ONE worker at a time.** Do NOT pre-plan a worker pipeline. The orchestration model is: dispatch one worker → evaluate findings (Worker Phase 2 Cross-Model Comparison) → reuse via `worker_send` or — when dead/done — scope the NEXT worker. Upfront multi-worker planning violates AGGRESSIVE REUSE (workers-3) and commits to a split before Worker Phase 2 findings justify it.

- Which gap (from Step 3) does this first worker close?
- Is there an alive worker with overlapping context already? → prefer `worker_send` over a new spawn (see workers-3 AGGRESSIVE REUSE). Otherwise → fresh `worker_spawn`.
- Abstract task, relevant files, Reference Files to follow.
- Subsequent workers get scoped LATER, after the current one completes or dies.

### Step 5 — Deliverables & KPIs

Define task-level deliverables with measurable completion criteria — NOT per worker. A single worker may close one deliverable or several (via follow-up `worker_send`). Worker-to-deliverable mapping emerges as the task runs.

- Each deliverable: WHAT is done, HOW to verify (test command, file exists, output matches)
- Plan file MUST include a Deliverables section with KPIs

**Present in chat for each deliverable:**
- What will be built/fixed
- How Opus verifies it (run tests, MCP call, check output) — code review does NOT count as verification
- How the user verifies it as final quality gate
- All affected file categories (src/, decisions/, dev/, docs)
- The FIRST worker's task + whether it's a fresh spawn or a reuse via `worker_send`

🛑 STOP — Ask for remarks before proceeding to IMPLEMENT.

---

## IMPLEMENT — Worker Phase 1: Dispatch — Task + Verständnis erfragen

**Pattern:** Give the worker the abstract task. Ask HOW they would solve it BEFORE they implement.

### Task Complexity → Plan or Go

**Umfangreiche Tasks (multi-file, Architektur, unklarer Scope):** Worker MUSS erst Plan vorlegen. Prompt enthält: "FIRST: Read files. Then describe your plan BEFORE implementing."

**Straightforward Tasks (bekannter Fix, eine Datei, klarer Scope):** Worker kann direkt implementieren. Prompt enthält: "Read files, implement, commit."

**Entscheidungskriterium:** Wenn Opus den Fix in 1-2 Sätzen beschreiben kann → straightforward. Wenn Opus selbst nicht genau weiß was sich ändern muss → Plan-Pflicht.

### Spawning

1. Write prompt to `/tmp/spawn-worker-<project>-<name>.md`
2. `worker_spawn(name, prompt_file, project_path, model, worktree)`
3. IMMEDIATELY set background timer: `Bash(command="sleep N && echo 'check'", run_in_background=true)`. See workers-2.md § Timer & Polling Flow for canonical timer defaults.
4. **Sequential spawn for cache-sharing:** When spawning multiple workers of the same model family (both Sonnet), dispatch them SEQUENTIALLY in separate response turns — not parallel in the same tool-call block. Worker 2's REQ#1 can only inherit cache from Worker 1 if Worker 1's first request completed before Worker 2's spawned.

### Prompt Structure

The prompt describes WHAT, the worker figures out HOW. Every prompt must match exactly what was agreed with the user — no "while we're at it" extras, no additional variables the user didn't ask for.

**MUST include:**
- The task described abstractly — what is the problem, what is the desired outcome
- Which files/directories are relevant
- Worktree path (see Worktree Rule below)
- Explicit negative scope: "Do NOT add features/improvements beyond the listed deliverables"
- "You are a WORKER."
- The investigate-report-stop pattern:
  > "FIRST: Read the relevant files. Describe your findings on root cause / approach and WHY. Then STOP and wait for Go before implementing."
- **Completion Checklist** — task-specific verification items the worker outputs when done
- **STOP Gate sentinel** as the LAST block in the prompt (see below)

**MUST NOT include:**
- Exact code to write (the worker figures that out)
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

**Split / Refactor tasks — instruct the worker to drop sections that don't apply.** When splitting an existing document or refactoring across N target files, workers tendentially copy ALL sections from the original into every target — including sections without legitimate content in the new scope (Quellen with no real sources, Evidenz with no layer-specific data, Offene Fragen that don't apply). Add to the prompt: "For each section in the target file, verify it has legitimate content in the new scope. If a section would be empty, stub-only, or factually inappropriate — leave it out entirely. Do not carry sections blindly from the original."
**Multi-File Bug — caller + callee both required as investigation target.** When a bug involves two-stage rendering (Pane → Format → Render), shared state across modules (Handler writes / Renderer reads), or any caller→callee data flow, the worker prompt MUST list BOTH files as investigation targets — not only the "fix-target" file. Reading just one side misses the contract mismatch.
### STOP Gate Enforcement (CRITICAL)

A single "STOP and wait for Go" line buried in a section header is UNRELIABLE. Completion-biased workers will read past it and proceed straight to implementation. To make the Phase 2 gate hold:

1. **Repeat the STOP** — state it once at the top of the instructions AND as the absolute last line of the investigation step.
2. **Use a sentinel block** — format the final STOP as a visually-prominent block at the very end of the prompt:
   ```
   ### 🛑 STOP HERE — DO NOT PROCEED WITHOUT GO
   Report your findings. Wait for "Go" from Opus before starting implementation.
   Do NOT run any Edit, Write, or Bash tool calls that modify files until Go is received.
   ```
3. **Forbid tool classes, not just "don't implement"** — workers interpret "don't implement" loosely. Be explicit: "Do NOT run Edit/Write/Bash file-modifying calls" is unambiguous.
4. **Place the sentinel AFTER the Completion Checklist** — the last thing the worker sees must be the stop gate.

### Worktree Rule (NON-NEGOTIABLE)

**ALWAYS spawn workers with `worktree=true` (the default).** This includes pure research workers that only read files or call MCP tools without editing code.

`worktree=false` creates the worker session in the SAME `~/.claude/projects/` directory as the main session. This causes the monitor Token-Pane to pick up the worker JSONL as "newest session" and display worker data instead of the main session.

**Tell the worker WHERE they are.** Every worker prompt MUST explicitly state the worktree path and frame it as their workspace:

> Your worktree: `<project>/.claude/worktrees/<name>/`
> This is your workspace — read, edit, test, and commit here. Do NOT touch files outside this path unless explicitly instructed.

Without this, workers sometimes navigate to the main project tree (same repo, different checkout) and edit there — edits land on the wrong branch and are lost on merge.

**Only exception:** Worker MUST edit gitignored files that don't exist in the worktree → `worktree=false`. This is rare.

