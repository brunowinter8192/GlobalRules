# 2026-05-07 — searxng: Current-State Claims Must Be Code-Verified

## ~/.claude/shared-rules/global/verify-before-execution.md → "Verify Inputs (Execution)" section

**Current-State Claims (NEW subsection — append after the Numeric-Values rule):**

When proposing a fix or improvement that says "the current behavior is X, should be Y" — VERIFY that "X" is actually the current state by reading the current source code, CLI help output, or config file IN THIS SESSION. NEVER claim the current behavior from memory, training knowledge, or assumption based on what an API "usually" defaults to.

The pattern that breaks: "I know API X has default behavior D. We presumably use D. We should change to D'." — but the "presumably" step skips reading what we ACTUALLY use. Often we already use D' (then the lever is empty work) or we use a third value entirely (then the proposed fix is wrong-target).

**Test before proposing a fix:** am I about to say "we currently use X" or "the default is Y" without having read the relevant source/config/help OUTPUT-of-this-session? If yes → STOP, read the artifact first.

The cheapest verification is usually trivial:
- Code state: `grep "<param>" src/<engine>/<file>.py` or one Read tool call
- CLI state: `<cmd> --help` (1 Bash call, returns full surface)
- Config state: `cat .env.example` or the live `.env` if applicable

This rule is the same principle as the Numeric Values rule — extends from "specific numbers" to "current implementation state".

Concrete failures (2026-05-07, searxng session):

1. **Stack Exchange `sort` lever** — claimed our SE engine used `sort=activity` (the API default) and proposed switching to `sort=relevance` as a project improvement. Worker investigation revealed `sort=relevance` had been in `src/search/engines/stack_exchange.py` since the initial engine commit (`78b2ce0`, "feat: drop HN, add Stack Exchange engine"). Lever was empty work; only caught because the worker's Plan-Pflicht had it read the code before implementing. One pre-claim `grep "sort" src/search/engines/stack_exchange.py` would have prevented the false claim entering the lever-inventory at all.

2. **RAG-CLI delete pattern** — claimed surgical chunk deletion required full re-indexing of the source dir. Reality: `rag-cli delete --collection X --document Y` already supported surgical chunk-only deletion (just lacked the source-MD-removal flag). One `rag-cli delete --help` would have shown the existing capability before proposing a re-index workflow as audit strategy.

Both: assumed an IST-state from training-knowledge or guesswork without checking the actual artifact in this session. Augments existing `verify-before-execution.md` rules on numeric values, file paths, etc.
