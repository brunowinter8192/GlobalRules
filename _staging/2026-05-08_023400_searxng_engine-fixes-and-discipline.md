# 2026-05-08 — searxng: engine-fix session, discipline lessons

## ~/.claude/shared-rules/global/tool-use.md → Hard Rules Rule 4 (Context window hygiene) — SHARPEN existing rule

The rule already prescribes the redirect-then-extract pattern (`> /tmp/file 2>&1` then `tail -20 /tmp/file`). Real-world failure 2026-05-08: I deviated by inlining `| head -400 > /tmp/file` which burrowed the upstream output into head's buffer for 90s of user waiting. The existing rule's example is correct; what's missing is one explicit anti-pattern line.

**Proposed sharpening — add ONE bullet to the existing list at line 82-84 area:**

```markdown
- NEVER chain `| head -N` (or `| tail`) inline before a file redirect — head BUFFERS until N lines OR EOF when stdout is a file/pipe; if upstream produces fewer than N lines, the file stays 0 bytes until upstream finishes, looking like nothing happened. Pattern is always: redirect to file FIRST, then extract from file with head/tail/grep.
```

This is a one-liner addition. No new section, no new heading. The example in the existing rule already covers the right pattern.

## ~/.claude/shared-rules/opus/workers-2.md → Phase 1 Pre-Spawn Shared-File Conflict Check section

Extend the existing pre-spawn check to cover NEW worker spawns from a now-stale dev:

Current rule wording covers only `worker_send` follow-ups across merges. The new-spawn case needs analogous coverage:

```markdown
**Pre-Spawn Branch Sync (when spawning AFTER recent merges):**

Worktrees branch from the dev tip AT SPAWN TIME. If worker A merged into dev at 14:00, and you spawn worker B at 14:30, B's worktree is current. But if you spawn worker B at 13:50 (BEFORE A's merge) and A merges at 14:00, B's worktree is now stale relative to dev. When B finishes and merges back, `git diff B dev` will look like B reverts A's changes — the diff is misleading because git's 3-way merge can resolve cleanly using merge-base, but only when B and A modified DIFFERENT files. If they modified the same file, conflict.

**Rule:** before spawning a worker on a branch that may have just received a merge:
1. Check `git -C <project_root> log dev --oneline | head -3` — confirm the latest commit timestamp
2. If a recent merge (<10 min) included files in the new worker's likely scope: defer the spawn until after the new worker has fetched the latest dev, OR pre-merge dev into the worktree first via `git -C <worktree> merge dev`
3. If unsure: spawn anyway. 3-way merge handles disjoint-file conflicts. But check the post-merge diff — `git -C <project> log -- <touched_file>` should show both sets of changes.

Lucky outcome 2026-05-08: openlibrary worktree branched before ss-fix merged. openlibrary's `git diff dev` showed semantic_scholar.py reverted (because openlibrary had OLDER copy than dev's new copy). 3-way merge resolved correctly because openlibrary didn't modify SS file. If openlibrary had modified SS file, this would have been a real conflict. Future worker dispatches should not rely on luck — explicit branch-state-check before spawning multiple workers in sequence.
```

## ~/.claude/shared-rules/opus/workers-2.md → "While Workers Run" section

Add a sub-rule on letting workers self-diagnose before override:

```markdown
**Mid-Task Worker Self-Correction Window:**

When you observe a worker hitting a problem (filesystem error, partial output, anomaly in capture), do NOT immediately send a corrective `worker_send`. Workers have local context — uncaught exceptions, recently-read files, in-progress hypotheses — that you cannot see. They will often diagnose and self-correct within the same response if given a few seconds. If after capturing a worker mid-investigation you see them already analyzing the issue ("I see the bug is X, fixing now..."), DO NOT interrupt with your own diagnosis.

**Rule:** if worker capture shows them in active diagnosis mode (text mentions "I see", "the bug is", "fixing", "let me check"), wait one more timer cycle (~60-90s) before sending a correction. If they self-correct: do nothing. If they remain stuck: then send the correction.

Real-world example 2026-05-08: cleanup-and-index worker hit zsh assoc-array bug with empty STEM. Opus spawned a STOP message asking what happened. Worker had ALREADY diagnosed the bug ("zsh löst ${STEMS[$fname]} in declare -A nicht auf wenn key Punkte enthält"), cleaned up, and was switching to Python script. Opus's STOP signal was queued, processed AFTER the self-correction. User feedback: "du musst den cleanup and index worker nicht kontrollieren das kann der schon alleine".

**Pivot:** read worker capture carefully BEFORE sending corrections. If their last 200 chars contain a hypothesis + remediation plan, the worker is mid-flight — let it land.
```

## ~/.claude/shared-rules/proj_searxng/engine_timeouts.md (NEW project rule)

Project-specific note on the multi-layer timeout architecture:

```markdown
# Engine Timeout Architecture

The searxng search pipeline has THREE coordinated timeout layers that must agree per engine:

1. **Inner engine timeout** — typically `httpx.AsyncClient(timeout=X)` for API engines, OR `tab.go_to(timeout=X)` for browser engines. Hardcoded inside `src/search/engines/<name>.py`.
2. **Outer asyncio watchdog** — `asyncio.wait_for(engine.search(...), timeout=Y)` in `_engine_with_timing` in search_web.py, value from `ENGINE_WATCHDOG_OVERRIDE.get(name, ENGINE_WATCHDOG_TIMEOUT)`.
3. **Rate-limiter cap** — `RATE_WAIT_TIMEOUT=5.0s` global, blocks new request when token-bucket exhausted.

**Coordination invariant:** for engines where layer 1 < layer 2, the inner timeout fires first → status becomes ERROR (httpx exception bubbles up) instead of TIMEOUT (asyncio.wait_for cancels). This obscures the diagnosis. Confirmed broken at one point: open_library had `httpx.AsyncClient(timeout=3.6)` while ENGINE_WATCHDOG_OVERRIDE['open_library'] = 6.0 — engine returned ERROR instead of OK with longer wait.

**Rule when adding a new HTTP-based engine:**
1. Set `ENGINE_WATCHDOG_OVERRIDE[name]` if engine has measurable latency variance (>50% of queries beyond 3.6s default)
2. Match `httpx.AsyncClient(timeout=X)` to the override value (X ≥ override)
3. Verify with one slow query: status should be TIMEOUT (from asyncio watchdog), not ERROR (from httpx)

**Rule when modifying ENGINE_WATCHDOG_OVERRIDE:**
- Grep `src/search/engines/<name>.py` for `httpx.AsyncClient(timeout=` AND `tab.go_to(timeout=` AND `WAIT_INTERVAL` AND `MAX_WAIT_CYCLES`
- Verify all are ≤ the new override value
- Otherwise either layer 1 (engine-internal) will fire first, masking the asyncio watchdog as ERROR.

**Caveat — pydoll non-cooperative cancellation (bead 7u5):** for browser engines using pydoll, `asyncio.wait_for` cannot reliably cancel CDP-WebSocket calls. The watchdog raises TimeoutError, but pydoll continues running until natural completion (observed: SS at 65s with 5.0s override). Status logs as TIMEOUT but search_ms far exceeds the watchdog. This means the wallclock guarantee is soft for browser engines. HTTP engines (httpx-based) cancel cleanly because httpx is properly async.

**Status semantics:**
- OK: results returned
- EMPTY: engine completed without exception, returned []
- TIMEOUT: asyncio watchdog cancelled (drop_reason "after Xs watchdog")
- ERROR: engine raised exception (drop_reason has stack-trace fragment)
- RATE_SKIP: token-bucket acquire exceeded RATE_WAIT_TIMEOUT (engine skipped for this query)
```
