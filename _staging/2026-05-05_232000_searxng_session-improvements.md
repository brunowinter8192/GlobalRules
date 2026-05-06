# 2026-05-05 — SEARXNG: Architectural-Cap Inspection, Architectural-Value Justification, Worker-Kill Branch-Deletion

## `~/.claude/shared-rules/global/verify-before-execution.md` → "Verify Inputs (Execution)" (new subsection)

Add a new subsection right after "Tool Extension":

**Architectural Cap Inspection (Wrap-Layer Read).** BEFORE proposing a timeout, watchdog, or any other cap that wraps a layer of code (e.g. `asyncio.wait_for(some_call, timeout=X)`), READ the wrapped layer end-to-end and identify every nested wait/sleep/blocking-acquire that the cap would unintentionally kill. A cap is only architecturally sensible if its budget covers the EXPECTED duration of the actual work, not waits for upstream resources. If a nested layer can sleep for longer than the proposed cap (e.g. a rate-limiter, a connection pool, a retry-with-backoff), the cap will fire on the wait and the wrapped call returns failure even when no real work was attempted — producing diagnostic data that conflates "stuck/slow" with "throttled/blocked".

The cheap test before proposing any cap: `grep -nE "await.*(sleep|acquire|wait_for|gather)" <wrapped_layer>` plus inspecting whatever those calls do downstream. If a nested wait is unbounded or longer than the proposed cap, redesign — either lift the wait outside the cap, change the cap value, or fail-fast the wait.

Concrete failure (2026-05-05): proposed `asyncio.wait_for(engine.search(...), timeout=8.0)` for searxng pipeline smoke without checking that `engine.search()` internally calls `await rate_limiter.acquire()` which sleeps up to 60s for token refill. First smoke run with the cap killed 14 of 30 queries because the rate-limit waits triggered the 8s watchdog on all 8 engines simultaneously — wasted ~14 minutes wall-time producing a half-empty unusable baseline. A pre-proposal grep of engine.py + read of rate_limiter.acquire would have surfaced this in 30 seconds. The eventual fix was an architectural refactor (rate-limiter moved out of engines into workflow-level acquire BEFORE the wait_for); could have been planned upfront.

## `~/.claude/shared-rules/global/verify-before-execution.md` → "Numeric Values in Reports / Analysis" (extend)

Extend the existing "Numeric Values in Reports / Analysis" subsection with a second paragraph covering ARCHITECTURAL VALUE PROPOSALS (not just report-content numbers):

**Architectural Numeric Value Justification.** When proposing a concrete number that controls behavior — timeout values, batch sizes, thresholds, rate limits, retry counts — the value MUST come from one of: (a) explicit empirical data citation (file, line, run output), (b) referenced constant in existing code/decisions, (c) explicit acknowledgment "this is a heuristic starting point, will be tuned by run output". NEVER derive a behavior-controlling value by mathematical conflation of unrelated dimensions (e.g. translating a RATE — N per minute — into a per-CALL latency budget — seconds — without empirical backing). Rate and per-call duration are independent dimensions; one does not imply the other.

Concrete failure (2026-05-05): proposed `engine_timeout=15s` based on "4 reqs/min ÷ 4 = 15s slot". This conflated request-rate (frequency) with per-call-latency (duration). User caught it explicitly. The actual evidence-based answer was visible in the smoke fanout data (bimodal distribution: healthy queries 1-5s, stuck queries 15-50s, with a clean gap in between) — any timeout in 5-12s was equally valid by the data. The value 15 had no empirical backing and was post-hoc rationalized.

## `~/.claude/shared-rules/opus/workers-3.md` → "Killing Workers" (extend)

Extend the existing "Killing Workers" subsection with a new bullet point about branch deletion:

**`worker-cli kill` silently deletes the worker's branch.** The kill command does three things: tmux session kill + worktree directory removal + `git branch -D <worker_name>`. The branch deletion is not configurable. If the worker's commits have NOT yet been merged into the main project (dev/main), the branch ref is gone after kill — but the commits themselves are still reachable via `git reflog` and the kill output line `Deleted branch <name> (was <hash>)` shows the hash for recovery via `git branch <name> <hash>`.

Rule: never invoke `worker-cli kill` on a worker whose commits are not yet merged. Verify before kill: `git -C <main_project> log <worker_branch>..main` should be empty (= worker branch is ancestor of main, all commits incorporated). If not empty: merge first, kill after. If kill happened by mistake on an unmerged branch: recover with `git branch <name> <hash_from_kill_output>` immediately — the hash is logged.

Concrete failure (2026-05-05): killed Worker 1 (named `dl9-v2`) before merging its branch into dev. The kill auto-deleted the `dl9-v2` branch with 5 commits of dl9-v3 architecture work. Recovery via `git branch dl9-v2 16fbd8f` from the kill output line worked, but cost 1 extra exchange and required reading kill output carefully. A pre-kill check (`git log dl9-v2..main` = should not be empty for unmerged work) would have caught this.
