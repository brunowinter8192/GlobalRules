# 2026-05-01 — searxng: dev-prod-discipline + architecture-mode anchor

## ~/.claude/shared-rules/opus/dev-prod-discipline.md (NEW FILE)

# Dev → Prod Discipline

**Dev MUST be ahead of prod or equal to prod, never behind.**

Never implement a change in prod that has not first been validated in dev. Empirical findings, refactoring decisions, performance optimizations — all must be tested in dev tooling first, with concrete evidence (smoke result, benchmark, eval). Only after dev validates: port to prod.

## Trigger

Any time a code change to prod files (`src/`, `cli.py`, `server.py`, equivalent) is on the table during PLAN. Before scoping the change as part of a worker task, ask:

1. Was this change tested in dev tooling, with empirical evidence?
2. If yes → port is sound, scope it.
3. If no → either:
   - drop the change from current scope, or
   - first build dev tooling that validates it, then come back

## Refactoring is not exempt

A "small refactor" of prod (e.g., remove a sleep, change a default, drop a constant) IS a change to prod that affects behavior. Even when it "obviously" makes sense, it must be dev-validated before going to prod. The dev tooling that exercises that prod path is the validation source.

## Direction is fixed

Dev → Prod, never Prod → Dev. If a worker collapses dev to use prod paths (e.g., refactor a smoke script to import from a prod engine), that breaks the boundary — dev loses its independent validation surface for future changes. Maintaining a separate dev path means dev can validate prod-bound changes BEFORE they land in prod.

## What does NOT trigger this rule

- Bug fixes for issues uncovered empirically by dev tooling DURING the current cycle. Example: dev's burst smoke uncovered a 60s atexit teardown bug in prod. Investigation + fix is appropriate in the same cycle because the empirical evidence (the bug measurement) was the dev validation.
- Feature additions that are net-new (no equivalent in dev) and that the user explicitly scoped. Example: adding a `search_batch` CLI subcommand is a feature, not a behavior modification of an existing dev-validated path.

## Concrete failure (2026-04-30, searxng MCP→CLI conversion)

Block B was scoped to include 4 sub-tasks: (a) Google port from dev to prod (dev-validated, OK), (b) decisions moves (cleanup, OK), (c) rate-limiter de-jitter (NOT dev-validated — `01_google_smoke.py` uses standalone pydoll, never exercises `src/search/rate_limiter.py`), (d) smoke refactor to use prod engine (collapses dev/prod boundary). User had to explicitly stop the worker mid-Phase B and reduce scope. Resulted in cherry-pick reset of two unrelated commits. Could have been prevented by checking each sub-task against the discipline upfront.

---

## ~/.claude/shared-rules/opus/workers-1.md → Phase 1 Step 2 — append "Architecture Mode Anchor" subsection

**Architecture Mode Anchor (before scoping new orchestration around CLI/server)**

Before scoping any worker that builds new orchestration around an existing CLI, server, or daemon, anchor the execution mode upfront:

- **subprocess-per-call** — each invocation is its own process, owns its own warmup (e.g., browser cold-start), no shared state across calls. Cheapest setup, highest per-call cost.
- **in-process batched** — one process handles N calls in sequence, shared state stays warm. Requires a batch API in the underlying tool.
- **daemon mode** — long-running process accepts calls via socket/IPC/HTTP. Most setup, lowest per-call cost.

Each mode has different cost characteristics — cold-start overhead, memory, isolation between calls, error blast radius. The cheapest implementation default is usually subprocess-per-call (just shell-out N times); the user's mental model often assumes a more amortized mode (one warm setup, multiple uses).

**Required:** identify the user's intended mode in PLAN before scoping the worker task. If the choice is non-obvious, ask. Don't assume.

## Concrete failure (2026-04-30, searxng burst smoke)

Built `02_burst_smoke.py` that ran subprocess-per-query against the production CLI (`searxng-cli search_web "<q>"` × N). Each subprocess started a fresh Chrome → 64s wall time per query (60s of which was an atexit CDP teardown bug). Empirical result: 5/30 OK in 15 minutes. Root cause: subprocess-per-query mode was assumed without anchoring against the user's actual model ("Chrome einmal starten, dann sequentiell N queries durch"). Once anchored to in-process batched (new `search_batch` subcommand, warm Chrome via singleton), wall time dropped to ~1.7 min for 30 queries with 28/30 effective OK. The architecture mode was the deciding factor; both other layers (selectors, stealth) were already correct.
