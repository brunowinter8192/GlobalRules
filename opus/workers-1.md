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

### External Resource Assessment

At key investigation moments the discipline is to STOP grinding hypotheses in-head and ask two questions IN ORDER:

1. **Do I need external information at all?** Often no — `<Project>-docs` RAG plus the project's own source code answer it. If the question is fully answerable from indexed docs + code, do NOT reach for any external tool.
2. **If yes — what EXACT fact/artifact do I need, and which source is authoritative FOR THAT question?**

The **source decides, not a default tool.** Match the question to where the answer actually lives:

- **The system/source itself** — authoritative for its own config and behavior. "Does site X have a sitemap, how deep?" → fetch `X/robots.txt` + the sitemap index directly. "What does API Y return for field Z?" → the vendor's API reference. First choice whenever the question is about a specific system's own behavior.
- **Vendor / framework docs** — API semantics, library behavior, config options.
- **General web** — behavioral / how-to questions, error-message lookups.
- **GitHub / code search** — "how do people actually implement X", real shipped patterns, reading a dependency's source.
- **Reddit / forums** — experiential reports and gotchas ("X blocks long scraping sessions", undocumented quirks).
- **Papers (arxiv)** — methodology, algorithms, academic grounding.

**No reflexive default.** Before reaching for ANY search tool, name the specific resource you need and justify the source.

**Trigger moments to run this assessment** — these are when in-head grinding is most costly:
1. Entering PLAN Step 2 on a new feature touching platform / framework / library surface.
2. After a failed iteration — "approach refuted" / "still doesn't work" / about to form hypothesis N+1.
3. A bug recurs after a RAG-aided fix attempt (second iteration on the same bug).

At each trigger: pause, run question (1) then (2). When external info IS needed, state — to the user, or in the PLAN — WHAT you need and WHERE you will get it BEFORE fetching, especially when more than one source could plausibly serve. Opus is the sole fetcher of external knowledge (§ External Knowledge — Opus Provides, Worker Implements); the worker never searches externally.

### Worker Model (NON-NEGOTIABLE)

Workers are ALWAYS **Sonnet**. NEVER Opus. Opus context is for orchestration only.

### Opus NEVER Edits Source Code (NON-NEGOTIABLE)

**ALL source code edits go through workers. ZERO exceptions.** This includes "quick fixes", "one-line changes", "obvious changes", and proxy/addon/config files. If it's a `.py`, `.sh`, `.js`, `.ts`, or any source file — WORKER.

Files Opus may edit directly: automation files (`.claude/rules/`, `.claude/commands/`) and documentation (`DOCS.md`, `decisions/*.md`, `decisions/OldThemes/**`). Source code stays worker-only. Documentation authorship splits by content origin — see § Documentation Authorship below.

**Opus does directly:**
- Verification (run tests, CLI calls, screenshots)
- Scoping, planning, rule edits
- `git` operations (commit, merge, branch)
- Reading/grepping source code for investigation
- Documentation derived from chat/discussion — `decisions/`, `OldThemes/`, `DOCS.md` (see § Documentation Authorship)

**Workers do:**
- ALL source code edits — no exceptions, not even "just one line"
- `decisions/` + `OldThemes/` updates that document the worker's OWN implementation/test/investigation work; dev script creation
- ALL dev script execution (stress tests, benchmarks, evals) — Opus does NOT run `./venv/bin/python dev/...` via Bash


**Post-merge fix flow:** Bug found after merge → `worker_send` to the still-alive worker. If worktree is stale → spawn new worker from current `dev`. NEVER edit source files yourself.

**Scope:** this rule applies WITHIN the current project. Cross-project edits follow the Worker Project Scope rule below.

### Documentation Authorship — Opus vs Worker

`decisions/` and `decisions/OldThemes/` are NOT source code — Opus may write them directly. The split is by **content origin**, not file type:

- **Chat/discussion-derived → Opus writes directly.** Research synthesis, decision rationale argued out in the Opus↔user chat, alternatives weighed in conversation, external-knowledge findings Opus gathered (RAG / vendor docs / web). When the "meat" already lives in the chat, Opus holds the full context — routing it through a worker turns the worker into a transcriber, adds latency, and loses fidelity. Write it directly.
- **Worker-implementation/test-derived → the worker writes.** IST updates after the worker changed `src/`, eval/benchmark/probe results, per-phase investigation logs from the worker's own runs. The worker holds that context and writes as part of its task or Phase-5 recap — keeping Opus context free.
- **Mixed sessions:** each side writes the part it produced — Opus the chat-synthesis, the worker the IST/test-result. Keep the content closest to where it was produced.

Source code stays worker-only regardless (§ Opus NEVER Edits Source Code). This subsection governs documentation only.

### Persistence Routing — Opus Decides Paths (NON-NEGOTIABLE)

Concerns split strictly (for work DISPATCHED to a worker): Opus owns routing decisions, worker owns content production. For chat-derived documentation Opus writes directly — see § Documentation Authorship.

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

**Opus is the SOLE interface to external knowledge sources:** RAG (`<Project>-docs`, `<Project>-reference`), books/papers, vendor/API docs, GitHub/code search, the web, and direct fetches from a source's own endpoints (robots.txt, sitemaps, status pages). Source selection follows § External Resource Assessment. Any formula, algorithm, method, constant, or external-source fact the worker needs — to PLAN or to IMPLEMENT — Opus extracts and provides IN THE PROMPT, distilled to the concrete content plus citation. The worker does NOT fetch external knowledge: no `rag-cli`, no external/code search, no web search, no reading external books/papers.

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
5. Session end: `git checkout main && git merge dev` to sync dev→main


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
2. **3rd-party library source** — e.g. tmux (`tty-keys.c`), mitmproxy addon hooks, any dependency whose behavior you'd otherwise guess at. Read the dependency's source directly (vendored locally, or via GitHub/code search).
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
- The worker does NOT fetch external knowledge: no `rag-cli`, no external/code search, no web search, no reading external books/papers/vendor docs. Any external fact, formula, method, algorithm, or 3rd-party/API semantic the worker needs is OPUS's to close BEFORE dispatch — Opus reads the source, distills the answer, and provides it in the prompt with the citation (see § External Knowledge — Opus Provides, Worker Implements).

**Part B — Mental Model Milestone (MANDATORY):**

Before proceeding to Step 4 (Worker Scope), Opus must be able to answer ALL of:

1. **What is the actual problem?** (not just symptoms)
2. **Which files/functions are involved and what do they do?**
3. **What are ALL the code paths the worker's task touches?** Have I READ each one in this session — not in a past session, not via RAG summary, not via DOCS.md description? If the worker's task involves instrumenting / modifying / interpreting behavior of function X, can I recite function X's branches and state mutations without re-reading?
4. **If a worker delivers "all done" with an INTERPRETATION of measured data, can I cross-check that interpretation against the source code that produced the data?** Specifically: are there alternative code paths that would produce the same measurement but support a different interpretation? If the worker says "data X means mechanism Y", do I know whether the source contains a mechanism Z that would also produce data X?

If ANY of these is NO → continue reading source code. Do NOT proceed to worker scoping. Root cause may still be unclear after Step 3 — that's OK. But Opus must understand the code surface well enough to EVALUATE worker output without re-doing the read at Phase 4 Review.

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
- How Opus verifies it (run tests, CLI call, check output) — code review does NOT count as verification
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

### Sequential Sub-Stage Decomposition

**Plan once for the whole; execute one stage at a time with per-stage Opus sign-off.** Applies when a task's implementation has interconnected, nested, build-on-each-other parts.

**Two-part discipline:**

1. **The PLAN is full and upfront.** The worker's Phase-2 report decomposes the ENTIRE task into ordered stages — the stages are interconnected, so the worker needs the whole picture to plan coherently. Do NOT fragment the planning. (This is the normal plan-first from § Task Complexity → Plan or Go; here the plan additionally names the stage sequence.)

2. **The EXECUTION is fed one stage at a time, each with Opus sign-off.** After convergence on the plan, dispatch ONLY Stage 1 ("implement Stage 1, commit, report"). Worker implements → commits → reports → Opus reviews that stage (Phase-4-light: diff + verify) → ONLY THEN dispatch Stage 2. Never "Go, build the whole plan." Each stage is a small, independently-committable, verifiable unit.

**A stage = one coherent committable unit**, sized so the worker finishes it in a bounded turn without a context blowout. Examples: the single-pass core before the multi-pass composition; the data extractor before its consumer; one file of a multi-file refactor; one pass migrated before the next.

**Interlocks with:**
- § Worker Phase 5 Recap (workers-2.md) — recap-after-every-stage already commits clean state per stage; this rule is its dispatch-side complement.
- § AGGRESSIVE REUSE / successor-from-checkpoint (workers-3.md) — staged commits are the checkpoints a successor resumes from.

**Trigger:** any substantial implementation — multi-file changes, algorithm/probe builds, refactors spanning several units, architectural ports. If the implementation has natural sequential stages OR would burn the worker through a large fraction of its context in one turn, stage it. Single-file known fixes (§ Task Complexity straightforward) need no staging.

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

