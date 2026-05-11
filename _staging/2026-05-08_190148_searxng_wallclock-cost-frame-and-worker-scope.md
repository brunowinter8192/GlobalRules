# 2026-05-08 — SEARXNG: Wallclock cost framing + worker-scope clarification

## ~/.claude/shared-rules/global/verify-before-execution.md → new section "Cost-Cheap Claims Need Cost-Frame"

Insert after the "Numeric Values in Reports / Analysis / Chat" section.

```
**Cost-Cheap Claims Need Cost-Frame.** When proposing a fix or test, "billig" / "cheap" / "low-cost" / "schnell zu testen" claims MUST be quantified relative to the dimension that matters to the user — usually wallclock cost in user-facing latency, NOT internal effort or LOC.

A proposal "+15s wait for the /sorry/ page to auto-solve" framed as "billig" because it's small in code (single sleep) is misleading: the user-facing cost is +15s on every blocked academic query, which on top of a 7s baseline is a 3× wallclock increase — catastrophic UX. The correct framing: "cost in code: minor; cost in wallclock: +15s per blocked query, +Y% on the academic-query latency budget".

Before describing a proposal as cheap/billig: enumerate cost dimensions (code LOC / wallclock per call / API budget / context budget / IP reputation / rate-limiter spend) and quantify each. If quantification reveals an expensive dimension that "cheap" was hiding, drop the framing or qualify it explicitly: "cheap in X, expensive in Y".

This is especially relevant for time-domain proposals (delays, retries, polling loops, increased timeouts, sleep-based fixes) where wallclock is the natural cost dimension and is often invisible from the code side. The default assumption — that adding a sleep is "free" — is wrong wherever the call is on a user-facing latency path.
```

## ~/.claude/shared-rules/opus/workers-3.md → "Killing Workers" section, add subsection on scope-of-"all"

Insert near the existing "NEVER kill without checking `worker_status` first" rule.

```
**"Alle Worker" / "kill all workers" — scope is THIS SESSION, not project-wide.**

When the user says "alle Worker" / "all workers" / "kill all" without qualifier, the default scope is the workers Opus spawned in the CURRENT session. NOT all workers from prior sessions, NOT cross-project workers, NOT workers attached to other developer's tmux state.

Before issuing `worker-cli kill <name>`:

1. Cross-reference the worker name against THIS session's spawn history. A worker not spawned this session = out of scope.
2. If a worker in the same project but from a prior session matches, ask: "Worker `<name>` is in this project but I didn't spawn it this session — kill that one too, or only this-session workers?"
3. Cross-project workers (different repo, different `worker-cli list` project_path column) are never killed without explicit confirmation that names that worker.

The cost of a stray kill is large: the worker's worktree branch and registry entry are deleted irreversibly. Uncommitted state is gone. If it was someone else's session-in-progress, that session is broken.

The cost of asking is one extra exchange.

Source: 2026-05-08 session — Opus killed `cleanup-crypto-papers` (a prior-session worker in the same project) on a "alle worker außer den eben gestarteten" instruction. User clarified after-the-fact: scope was this-session-only. Worker was lost. The instruction was reasonably interpretable both ways; the safer default is the narrower scope, ask if unsure.
```
