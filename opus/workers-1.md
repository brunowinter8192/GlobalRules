# Workers

See `global/tool-use.md` § Worker CLI for full command reference.

## Core Rules

### Cross-Model Verification

**Convergence** on root cause / approach → Go implement.
**Divergence** → at least one side is wrong → iterate investigation, NOT implement.

Applies to EVERY task, not only "unclear root cause" cases. Even when Opus believes they know the answer, the worker's independent investigation IS the verification.

### RAG-First on Any Project Question (NON-NEGOTIABLE)

**Gate:** before composing ANY answer to a user question about the project — and before ANY Read/Bash/Grep/git/find exploration that supports such an answer — run `rag-cli search_hybrid` on `<Project>-docs`.

`<Project>-reference` is NOT part of this gate. On-demand only, when the question concerns external system behavior.

Convention: `~/.claude/shared-rules/global/documentation.md` § RAG Collection Layers.

The trigger is **the user asking about the project**, not "code exploration" specifically. Status questions, orientation questions, "what did we decide", "is X still current" — these are the highest-value RAG queries, NOT edge cases of the rule.

**Trigger phrases that MUST fire RAG before any answer** (non-exhaustive, German + English):

- "wo stehen wir" / "where are we" / "Stand" / "Status" / "Überblick" / "recap"
- "was haben wir entschieden zu X" / "what did we decide about X" / "warum X so" / "why is X this way"
- "wie ist X aktuell" / "is X still current" / "passt Y noch" / "is Y still right"
- "haben wir Z schon" / "do we have Z" / "gibt es schon W" / "is there a W"
- "was ist <component>" / "what is <component>" / "wie funktioniert X" / "how does X work"
- "wann haben wir X gemacht" / "when did we do X" — even history questions hit decisions/OldThemes, not git

If the user's question matches any of these patterns → STOP, run BOTH RAG queries FIRST, then compose the answer using the hits as the primary source.

**Trigger PER TOPIC.** Every new topic / new user question gets fresh RAG. Pivot from Topic A to Topic B mid-session = fresh queries on B. Don't reuse hits from earlier in the session for a different question — re-query.

**Bugs count as topics.** Before any source-read on a crash / exception / "X doesn't work" report — RAG-search BOTH collections for the symptom. Past gotchas are indexed and frequently hit. Pass any matching prior finding to the worker as a hint, do NOT let the worker re-derive it.

**Forbidden Proxy Sources for project-state answers.** The following are NEVER substitutes for RAG when answering "what's the current state / where do we stand / what did we decide":

- `git log` / `git diff` — shows activity, NOT the documented state. Code that landed may have been superseded in decisions/ a day later.
- `gh-cli list_issues` alone — issues are entry-points pointing at sources, not the sources themselves. Read the Source-Inventory, then RAG on the topic.
- `find dev/ -name "*reports*"` / mtime checks — tells you when files changed on disk, NOT whether the report reflects current prod config.
- `ls -lt` over any directory — same problem.

These tools are valid SUPPLEMENTS *after* RAG has produced the primary answer. They are FORBIDDEN as the primary source for status/orientation/decision questions. If you find yourself reaching for `git log` to answer "wo stehen wir" — STOP, RAG first.

**Pre-Answer Self-Test (every user-facing answer about the project):**

> "Before I write this paragraph — did I query `<Project>-docs` with at least one topic-relevant phrasing in THIS session for THIS question?"

If no → STOP, query first. Even when the question feels trivial. Even when you "remember" the answer from earlier work.

**Escalation chain:**

1. `rag-cli search_hybrid "<query>" <Project>-docs`
2. On miss: reformulate, ≥ 2 phrasings before "no hit" valid
3. `rag-cli read_document <coll> <doc> <chunk_index> --before N --after M` on partial hits, not re-query
4. Only then: direct-read on indexed file, OR supplement with git/find AFTER RAG-derived answer is composed

(Reference layer: separate trigger, `rag-cli search_hybrid "<query>" <Project>-reference`, not part of this chain.)

**Source code is NOT indexed.** Direct Read/Grep on `src/*.py`, `dev/*.py`, `*.sh`, config files for mechanical questions. RAG on indexed layers FIRST, then targeted source reads for the gap.

### GitHub-Search Counter-Check (NON-NEGOTIABLE)

External-pattern verification via `gh-cli-search` skill fires at TWO trigger points:

1. **BEFORE dispatching any worker on a new feature / new architectural problem** — when the task touches platform APIs, framework conventions, library integrations, or any "how do people normally do X" surface. Cost: 1–3 `gh-cli search_code` / `search_repos` calls (~30 seconds). Gain: convergent patterns from real shipped code that prevent hypothesis-grinding.
2. **AFTER any failed iteration on a hard problem** — when one investigation cycle ended with "approach refuted" / "still doesn't work" / "let's try another hypothesis". BEFORE the next hypothesis is formed in-head, run gh-search. If two independent repos converge on the same pattern, that pattern wins over any in-head hypothesis.
3. **BUG RECURS AFTER RAG-AIDED FIX ATTEMPT** — bug → RAG hint → fix → bug still there. Second iteration on the same bug. Before another hypothesis: gh-search the symptom + API surface. No more in-head trial-and-error past iteration 2.

**Hard rule:** when entering PLAN Step 2 (Prep Investigation) on a new feature touching platform/framework/library surface OR when about to formulate a new hypothesis after a failed iteration → activate `gh-cli-search` skill (search strategy is in the skill itself), then proceed with the PLAN.

**Self-test before any worker-dispatch on a new architectural problem:** "Did I run a gh-cli-search on the platform/framework keyword in this session's topic?" If no → STOP, run the search first.

**Self-test before formulating hypothesis N+1 after iteration N failed:** "Did I gh-cli-search the failure pattern (specific API, observed symptom) before guessing again?" If no → STOP, search first.

### Worker Model (NON-NEGOTIABLE)

Workers are ALWAYS **Sonnet**. NEVER Opus. Opus context is for orchestration only.

### Opus NEVER Edits Source Code (NON-NEGOTIABLE)

**ALL source code edits go through workers. ZERO exceptions.** This includes "quick fixes", "one-line changes", "obvious changes", and proxy/addon/config files. If it's a `.py`, `.sh`, `.js`, `.ts`, or any source file — WORKER.

The ONLY files Opus may edit directly: automation files (`.claude/rules/`, DOCS.md, `.claude/commands/`).

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

### Persistence Routing — Opus Decides Paths (NON-NEGOTIABLE)

Concerns split strictly: Opus owns routing decisions, worker owns content production.

**Opus does (BEFORE dispatch — part of PLAN, not IMPLEMENT):**

1. **OldThemes folder.** RAG-search `<Project>-docs` to check whether an OldThemes folder for the topic already exists. If yes → reuse the exact slug. If no → decide the slug name (matching project naming conventions). Do not delegate slug invention to the worker.
2. **decisions/<step>.md files.** RAG-search `<Project>-docs` to identify which decision files the upcoming work touches (may span multiple pipeline steps — list ALL of them, not just the most obvious).
3. **New folders / new files.** Decide explicitly whether the task creates a new OldThemes topic folder, a new `decisions/<step>.md`, or only extends existing ones. If a new file/folder is needed and naming is non-obvious → ASK USER before dispatch.
4. **Pass exact paths in the worker prompt.** Worker prompt names full paths: e.g., "Write Phase A.1 narrative to `decisions/OldThemes/<exact-slug>/A1.md`. IST updates after src/ change go to `decisions/<step>.md` and `decisions/<other>.md`." No placeholders, no "the worker decides".

**Worker does:**
- Reads the paths Opus provided
- Writes content to those paths
- Does NOT use `rag-cli`
- Does NOT pick new folder names
- Does NOT decide where narrative lives

If a worker invents a path mid-task, that's an Opus rule violation (incomplete prompt) — not the worker.

**Applies to:** OldThemes narrative paths, decisions/ IST update paths, new `dev/<area>/` subdirectory naming, any artifact placement decision.

### External Knowledge — Opus Provides, Worker Implements (NON-NEGOTIABLE)

**Workers own SOURCE CODE. Opus owns everything external.** A worker reads, writes, and reasons about source code. Anything outside the source tree — external knowledge, theory, formulas, methods, vendor/API semantics, library behavior documented elsewhere — is NOT the worker's surface.

**Opus is the SOLE interface to external knowledge sources:** RAG (`<Project>-docs`, `<Project>-reference`), books/papers, vendor/API docs, gh-cli-search patterns, web. Any formula, algorithm, method, constant, or external-source fact the worker needs — to PLAN or to IMPLEMENT — Opus extracts and provides IN THE PROMPT, distilled to the concrete content plus citation. The worker does NOT fetch external knowledge: no `rag-cli`, no `gh-cli-search`, no web search, no reading external books/papers.

**The worker's independent investigation is scoped to SOURCE CODE** — the project's own code (and, where it directly integrates one, the source of a library it calls, to get the API/behavior right). That is the cross-model verification surface: the worker reads the code independently, reasons about the approach, Opus compares. Method/formula CORRECTNESS is Opus's responsibility — Opus reads the external source in PLAN Step 2/3 and provides the distilled result with its citation in the prompt.

**If the worker hits an unanticipated external-knowledge need mid-task** (a formula/fact/API-semantic Opus did not provide) → it STOPS and asks. Opus fetches and provides. The worker never goes and fetches external knowledge itself.

The split: SOURCE CODE = worker surface (read + write + reason). EXTERNAL THEORY/KNOWLEDGE/FORMULAS = Opus surface (Opus reads, distills, hands over).

### Worker Project Scope

**Workers are spawned only for coding tasks IN THE CURRENT PROJECT** (`pwd` at session start). Cross-project edits Opus does directly — no carve-outs. Size of change doesn't matter. Trigger to spawn a worker is "this is the current project"; anywhere else → Opus directly.

If a cross-project task feels too large for one session, pivot the session to that project rather than spawn a worker without project context.

**Worktree rule holds for the current project:** workers always go into a worktree, no exceptions. See § Worktree Rule.


### Dev-Branch Workflow

Workers merge onto `dev`, not `main`. Session end: `git checkout main && git merge dev`.

1. Session starts on `main` → `git checkout -b dev` (or switch to existing)
2. **Branch-State-Check when switching to existing dev (MANDATORY):** `git -C <repo> log dev..main --oneline | head -10` — if non-empty, dev is BEHIND main. Workers would spawn on stale code. Resolve before spawning: rebase dev onto main (clean when no dev-only commits) OR merge main into dev (preserve dev topology). Stay on stale dev only with explicit user OK.
3. Workers spawn (worktrees branch from `dev`)
4. `worker_merge` merges into `dev`
5. Session end: `dev_sync` MCP tool to sync dev→main


### Pre-Spawn Shared-File Conflict Check

Worktrees branch from the last COMMIT — uncommitted changes are NOT visible to the worker. BEFORE dispatching: commit changes, or tell the worker explicitly NOT to modify locally modified files.

---

## Session Cycle

### Session Start (MANDATORY)

→ read open issues for the current project's repo (`gh-cli list_issues brunowinter8192 <repo>`).

### Position Indicator

When in a PLAN/IMPLEMENT cycle, every response starts with a position indicator:

- `📋 PLAN — Step 1: Session Scope`
- `📋 PLAN — Step 2: Investigation`
- `📋 PLAN — Step 3: Gap Analysis`
- `📋 PLAN — Step 4: Worker Scope`
- `📋 PLAN — Step 5: Deliverables & KPIs`
- `🔨 IMPLEMENT — Worker Phase 1 (Dispatch)` / `Worker Phase 2 (Evaluate)` / `Worker Phase 3 (Go)` / `Worker Phase 4 (Review)` / `Worker Phase 5 (Recap)` / `Worker Phase 6 (Merge)`

Outside an active cycle (chat, casual response, status answer): no indicator needed.

### Cycle Overview

**PLAN** — Opus understands, scopes, defines deliverables. NO worker dispatched yet.
**IMPLEMENT** — Workers active. Each worker runs through Worker Phases 1-6 (Dispatch → Evaluate → Go → Review → Recap → Merge).
**RECAP** — Session end (separate `recap` skill).

---

## PLAN Phase — Understand & Scope

Sequential steps. After each step: present findings, wait for remarks, then proceed.

### Step 1 — Session Scope

Repeat what the user wants in your own words.

🛑 STOP — Ask for remarks.

### Step 2 — Prep Investigation (RAG → Source Identification → Source Read)

Opus builds its OWN mental model — NOT to be confused with Worker Phase 2 cross-model investigation. Two independent investigations are the whole point of the orchestration model:

- **PLAN Step 2 prep (here):** Opus builds an own mental model from indexed sources AND the actual source code. Cannot be delegated — if Opus has no model, Opus cannot evaluate worker findings later.
- **Worker Phase 2 cross-model (workers-2):** the dispatched worker reads files in the worktree independently, reports findings. Opus compares the two models. Convergence → Go; divergence → iterate.

Delegating the PLAN-Step-2 prep to an "Investigation Worker" collapses the two sides into one — you lose the independent second model, and with it the verification power.

**Three-stage workflow, sequential. Each stage feeds the next.**

#### Stage 1 — RAG (find WHAT and WHICH)

Read the Issue (Source-Inventory + Resume hint), then run RAG searches per § RAG-First on Any Project Question above. Two collection layers for projects with `.rag-docs.json`:

- `<Project>-docs` — internal docs: decisions/, DOCS.md, OldThemes (current state + discussion trail)
- `<Project>-reference` — external papers, vendor docs (when maintained)

**Stage 1 purpose: identify the topic landscape and produce a read-list of source files for Stage 3.** RAG indexes summaries, decisions, and discussion trails — NOT source code. A RAG hit that says "acquire() with backoff support" does NOT carry the actual code paths (e.g. "acquire() has TWO `await asyncio.sleep` branches, one for backoff and one for token-bucket-cap"). That lives only in the function body.

**Stage 1 output:** a concrete read-list — `src/<package>/<module>.py` plus 3rd-party library files if relevant. Not "modules around X" — actual file paths.

#### Stage 2 — Source Identification (which files MUST be read)

From the RAG hits, extract every src/ (and 3rd-party-library) file the worker will touch, instrument, modify, or whose behavior the worker's task depends on. Add adjacent files where the worker's interpretation will live — e.g. if the worker will instrument `rate_limiter.py`, the engine modules that CALL `rate_limiter.backoff()` are also on the list, because the interpretation of "where backoff comes from" depends on them.

**Heuristic:** if a worker's deliverable will interpret measurements from function F, file containing F is mandatory-read; files containing every caller of F are mandatory-read; files containing every state mutator of F's state are mandatory-read.

#### Stage 3 — Source Read (build the actual mental model)

Read every file on the read-list. Not skim — READ. Every function, every state mutation, every code path the worker will touch. Mental-model contents that Step 3 Part B verifies are BUILT HERE, not in Stage 1.

**Quote-Test before leaving Stage 3:** for every function the worker will instrument, modify, or whose behavior the worker will interpret — can you, without re-reading, recite its branches? If `acquire()` has two `await asyncio.sleep` branches, you must know both before scoping a probe that observes when `acquire()` blocks. If you cannot quote file:line on the actual mechanism, Stage 3 is not done.

**Anti-pattern (the failure mode this stage prevents):**
- RAG returns a summary chunk + a worker is dispatched on its basis
- Worker reads the source themselves, builds an interpretation, returns findings
- Opus accepts the interpretation without reading the source — the entire chain becomes inference-stacked-on-inference
- The interpretation collapses under one factual challenge from the user, because Opus never had primary evidence to defend it
- Hours of work wasted on a probe-design that missed half the mechanism

**Stage 3 is non-negotiable when:** the task involves instrumenting, modifying, refactoring, or interpreting the behavior of a specific function or module. It can be SHORTER when: the task is purely additive (new file, new tool, no interaction with existing logic) AND no Worker output will interpret existing behavior.

**Present status quo to user after all three stages:**
- Which files/components are affected — with file:line citations from Stage 3, not just RAG summary phrasing
- Current state (IST) and why it matters
- Reference Files identified (Stage 2 read-list, marked as read)
- The actual code paths the worker's task touches (Stage 3 finding)
- Relevant dev/ scripts

🛑 STOP — Ask for remarks.

### Step 3 — Gap Analysis + Mental Model Check

Two parts:

**Part A — Gap Analysis:**

Produce a sources table: Component | Source | Coverage | Gap

**Explicitly enumerate ALL resource categories** — not only our own code:

1. **Our own code** — `src/`, `decisions/`, `dev/`, existing logs in `src/logs/` or `data/`
2. **3rd-party library source** — e.g. tmux (`tty-keys.c`), mitmproxy addon hooks, any dependency whose behavior you'd otherwise guess at. GitHub repos readable via the `gh-cli-search` skill.
3. **Vendor / API docs** — Anthropic API reference, Claude Code internals, etc. Often indexed in the `<Project>-reference` collection.
4. **Live data** — greppable proxy JSONL, session JSONL, existing reports. Structural evidence beats guessing at shape.
5. **Web / Reddit / arxiv** — last resort for behavioral questions not answered by source or docs.

For each resource: state WHICH question it answers. If no resource is listed for a question, the question is OPEN.

**Gap-closed means EVIDENCE, not plausible extrapolation.**

- Closed ✅ = "I have concrete evidence from resource X (file:line, grep count, doc quote, log entry) that answers question Y."
- NOT closed ❌ = "The existing code looks like it probably does Z, so the fix is probably W."
- Reading existing code is evidence about OUR code. It is NOT evidence about 3rd-party semantics (tmux button codes, mitmproxy hook order, Anthropic field shapes) — those need their own source.
- **"indexed" ≠ "answered":** Query the source, extract the answer, cite file:line or doc quote.

**Worker closes gaps ONLY at the project source code (Worker Phase 2 investigation):**
- The worker's investigation surface is the PROJECT's own source code — the files it will touch, instrument, modify, or whose behavior it interprets. That is the cross-model verification surface.
- The worker does NOT fetch external knowledge: no `rag-cli`, no `gh-cli-search`, no web search, no reading external books/papers/vendor docs. Any external fact, formula, method, algorithm, or 3rd-party/API semantic the worker needs is OPUS's to close BEFORE dispatch — Opus reads the source, distills the answer, and provides it in the prompt with the citation (see § External Knowledge — Opus Provides, Worker Implements).

**Part B — Mental Model Milestone (MANDATORY):**

Before proceeding to Step 4 (Worker Scope), Opus must be able to answer ALL of:

1. **What is the actual problem?** (not just symptoms)
2. **Which files/functions are involved and what do they do?**
3. **What are ALL the code paths the worker's task touches?** Have I READ each one in this session — not in a past session, not via RAG summary, not via DOCS.md description? If the worker's task involves instrumenting / modifying / interpreting behavior of function X, can I recite function X's branches and state mutations without re-reading?
4. **If a worker delivers "all done" with an INTERPRETATION of measured data, can I cross-check that interpretation against the source code that produced the data?** Specifically: are there alternative code paths that would produce the same measurement but support a different interpretation? If the worker says "data X means mechanism Y", do I know whether the source contains a mechanism Z that would also produce data X?

If ANY of these is NO → continue reading source code. Do NOT proceed to worker scoping. Root cause may still be unclear after Step 3 — that's OK. But Opus must understand the code surface well enough to EVALUATE worker output without re-doing the read at Phase 4 Review.

**Why points 3 and 4 are explicit:** the canonical failure mode is "RAG hit + Worker findings + plausible interpretation → dispatch fix → user picks one factual challenge → entire chain collapses because Opus had no primary source-code evidence backing the interpretation". Point 3 prevents the chain from starting (the read happens in Step 2 Stage 3 and is verified here). Point 4 prevents Phase 4 Review from rubber-stamping a worker interpretation that the source code does not uniquely support.

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
3. IMMEDIATELY set background timer: `Bash(command="sleep N && echo done", run_in_background=true)`. The `echo done` payload is literal — no descriptive text, no quotes. Only this exact form stays background; any variation is silently rewritten to foreground by `block_unauthorized_background` and then conflicts with parallel Bash in the same response. See workers-2.md § Timer & Polling Flow for canonical timer defaults.
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

**Empirical Investigation tasks — name the OldThemes topic slug, don't repeat the structure.** When the worker prompt is for an architectural alternative, library swap, or trial-and-error verification (dev/ probe territory per `worker-rules.md` § 5), the prompt names the OldThemes topic-slug (e.g., `decisions/OldThemes/menubar_nspanel/`). Worker handles the three-layer workflow on independent cadences — `dev/<area>/` scripts + DOCS.md (snapshots: edited only when probe-pattern invalidates or prod-mirror required), `decisions/OldThemes/<topic>/<phase>.md` per phase (live log: every phase, even when dev/ unchanged), `decisions/<step>.md` IST when prod state changes (IST edited FIRST, then src/, atomic commit). Do NOT enumerate any of this in the prompt — anchored in worker-rules.md § 5.

### STOP Gate Enforcement

To make the Phase 2 gate hold:

1. **Repeat the STOP** — state it once at the top of the instructions AND as the absolute last line of the investigation step.
2. **Use a sentinel block** at the very end of the prompt:
   ```
   ### 🛑 STOP HERE — DO NOT PROCEED WITHOUT GO
   Report your findings. Wait for "Go" from Opus before starting implementation.
   Do NOT run any Edit, Write, or Bash tool calls that modify files until Go is received.
   ```
3. **Forbid tool classes, not just "don't implement"** — "Do NOT run Edit/Write/Bash file-modifying calls" is unambiguous.
4. **Place the sentinel AFTER the Completion Checklist.**

### Worktree Rule

**ALWAYS spawn workers with `worktree=true` (the default)** — including pure research workers that only read files.

**Tell the worker WHERE they are.** Every worker prompt MUST explicitly state the worktree path and frame it as their workspace:

> Your worktree: `<project>/.claude/worktrees/<name>/`
> This is your workspace — read, edit, test, and commit here. Do NOT touch files outside this path unless explicitly instructed.

**Only exception:** Worker MUST edit gitignored files that don't exist in the worktree → `worktree=false`.

