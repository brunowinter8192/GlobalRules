# 2026-05-08 — Trading: RAG Infrastructure Cascade Lessons

## ~/.claude/shared-rules/global/tool-use.md → "Background Bash" section

Add anti-pattern note:

**Long-running commands the user should see in CC's UI MUST use `Bash(run_in_background=true)`.**
NEVER use shell-level `nohup ... &` inside a foreground Bash for tasks the user wants visibility into. The `nohup &` form runs the process in the OS background but does NOT register with CC's task tracking — the user sees nothing in the background-process panel and cannot inspect output via the CC UI.

`nohup &` inside a foreground Bash is acceptable ONLY for fire-and-forget tasks where the user explicitly does not need visibility (e.g., a cleanup script that logs to a file the user will read later). Default to `Bash(run_in_background=true)` whenever the user might want to monitor progress.

Concrete from 2026-05-08: re-indexing was started via `nohup ./venv/bin/python workflow.py index-dir > /tmp/out.log 2>&1 &` inside a foreground Bash. The process ran but did not appear in CC's background panel. User explicitly noted "ich sehe hier keine bash prozesse die du gestartet hast" and required restart with `Bash(run_in_background=true)`. Lost ~5 minutes.

## ~/.claude/shared-rules/opus/workers-1.md → "Phase 1 — Understand" / "Prompt Structure"

Add note in the prompt-structure guidance:

**Connection-management refactors require caller-lifetime audit in Phase A.**
When dispatching a worker for changes to DB connection management (timeouts, autocommit, lock acquisition, transaction lifecycle), the Phase A investigation prompt MUST explicitly include "audit ALL caller code for connection-holding lifetime patterns — does the outer connection stay open across loops? Does autocommit interact with lock acquisition? Are reads and writes mixed on the same connection?". Without this audit, layer-cascade bugs hide under the layer-fix.

Concrete from 2026-05-08 cascade: Phase 1 added timeouts to `db.py:get_connection()`. The fix worked in isolation but silently exposed a previously-hidden Layer 3 bug — `workflow.py` outer connection held read-locks across the indexing loop without autocommit, since psycopg2 default is `autocommit=False`. Pre-Phase-1, that bug was masked by infinite-wait semantics; post-Phase-1 it surfaced as `LockNotAvailable` after exactly 30s. Phase 4 (autocommit fix) shipped four hours later. If the worker prompt had asked "audit all caller code for transaction/lock-holding lifetime patterns", Phase A would have surfaced both the missing-timeout AND the missing-autocommit issues, single phase instead of four.

## ~/.claude/shared-rules/opus/communication-2.md → "Push-Back-Once, Then Dispatch" section

Add note:

**User signals "wir machen das jetzt zusammen / alles auf einmal" = full scope, not incremental.**
When the user signals all-in-one execution ("wir machen das jetzt", "in einem Aufwasch", "einfach komplett durch", "zusammen"), Opus MUST default to full scope and NOT unilaterally split-and-defer Phase 2 (or N) into a Bead. Splitting against this signal forces the user to push back to expand scope, costing exchanges.

If splitting genuinely makes sense (e.g., scope X requires architectural decisions before scope Y can be implemented), state the constraint EXPLICITLY: "Y depends on X being designed first; want me to design X now and then we decide on Y, or sketch both upfront and run Y after?". Do not present the split as a default.

Concrete from 2026-05-08: Opus initially scoped a worker for Phase 1 only (timeouts + signal handlers) and proposed Phase 2 (lock + status + dynamic-ports + watchdog) as a separate-session Bead. User had said "wir machen das jetzt" and "lass mit worker machen wir haben den einen am leben der kann anfangen" twice. Opus had to expand scope mid-task via worker-cli send, costing extra round-trips.
