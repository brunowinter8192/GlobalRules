# Features & Debugging Workflow

## Dev Scripts Before Production Code

**Problem:** Iterating on production code via live tool testing (MCP calls, API calls) produces no reproducible evidence and no regression coverage.

**Rule:** When testing reveals edge cases or bugs in production code:
1. **Reproduce in dev/ first** — create a dev script that reproduces the exact failure
2. **Prototype the fix in dev/** — implement and validate the fix in the dev script, test against edge cases AND baselines (control group)
3. **Only then modify production code** — with validated fix and reproducible test

**Why:** Dev scripts are reusable regression tests. Live MCP/API testing is ephemeral — you see the result once, then it's gone. Dev scripts produce timestamped reports that document what was tested, what passed, what failed.

**Concrete failure (2026-03-31):** Tested garbage detection improvements via live `scrape_url` MCP calls. Two edge cases discovered (consent-prefix, padded 404). Had to backtrack and create dev scripts (07, 08, 09) to properly investigate. The dev scripts revealed that Crawl4AI exposes `result.status_code` — something the live MCP test couldn't show because our wrapper discarded it. Dev-script-first would have found this immediately.

**Exception:** Simple feature verification (e.g., "does download_pdf return a file path?") can use live MCP testing. The rule applies when edge cases or unexpected behavior are discovered.

## MCP Plugin Testing Requires Sync Cycle

When testing MCP plugin changes live, the full cycle must complete before testing:
1. Commit changes on main
2. `plugin-sync.sh <name> <repo-path>` to update cache
3. Kill old server process: `ps aux | grep "<name>.*server.py" | grep -v grep | awk '{print $2}' | xargs kill`
4. `/mcp` to reconnect (starts fresh server with new code)
5. THEN test

**Never test MCP changes without completing this cycle** — the running server uses cached code in memory.
