# 2026-05-10 — Monitor_CC: Box architecture migration

## ~/.claude/shared-rules/global/verify-before-execution.md → new subsection "Pre-validate platform-specific subprocess commands in worker specs"

When writing a worker prompt that includes subprocess commands (`pgrep`, `ps`, `lsof`, `awk`, `sed` flag patterns) that will run inside the worker's worktree, empirically verify the EXACT command on the same machine the worker will run on BEFORE putting it in the spec. Cross-platform subprocess CLI behavior (especially macOS vs Linux) is not always intuitive and assumed-correct flags can silently be wrong, sometimes catastrophically.

Concrete: `pgrep <name>` (no flag) on macOS matches against the process `comm` field (executable basename, ~16 chars). `pgrep -f <pattern>` matches against the FULL argument list, which includes argv-from-prompt for any process whose argv contains the pattern as a substring — e.g. a Claude Code worker session whose tmux pane was spawned with a prompt mentioning `<pattern>` will be matched by `pgrep -f`. `pgrep -x <name>` matches the comm field exactly (no substring/prefix). For "find all running instances of binary X" use `-x`, never `-f`.

Failure 2026-05-10: Box-architecture spec for the RAG watchdog used `pgrep -f llama-server` in `_purge_orphans`. The worker followed literally. Live-probe at code review showed: `pgrep -f llama-server` matched the worker's claude.exe PID (because the worker's argv contained "llama-server" via the prompt text), `pgrep -x llama-server` matched only the actual binary (correct semantics for the use case). Continuous purge would have killed the worker via SIGTERM within one watchdog interval if not caught at review.

Rule: subprocess command-line invocations in worker specs MUST be empirically verified by Opus on the same machine where the worker will run BEFORE the spec is dispatched. The verification command is part of spec authoring, not a post-hoc review step. For platform-sensitive commands (`pgrep`, `ps`, `lsof`, `find -printf`, `xargs`), test BOTH the with-flag and no-flag variants and pick the one that matches the intended semantics.

## ~/.claude/shared-rules/opus/workers-2.md → "Smoke Tests" / "What you do NOT do" section

When a worker prompt instructs preservation of a running process ("do NOT kill X", "PID Y must survive"), the worker's smoke test in the Completion Checklist MUST include an actual liveness check (`curl -s -o /dev/null -w '%{http_code}\n' http://localhost:<port>/health` returning 200, or `ps -p <PID> -o etime` returning a row with non-empty elapsed time) for that specific process — NOT just URL-string-construction or import-resolves checks.

Constructing the URL via discovery, or importing a function that reads state files, does NOT prove the process is alive. The state file might exist with a stale PID. A function might return a URL that points to a dead port. The only proof of liveness is a live HTTP probe or a live PID check at the moment of the smoke test.

Failure 2026-05-10: box-cleanup-dynports worker (RAG Phase 5+6) had `_embedding_url()` in its Phase 6.2 smoke test. Helper returned `http://localhost:8081/v1/embeddings` correctly from the still-existing state file. Worker reported "PID 98342 (embedding) NOT killed" based on this URL-string check. Embedding had actually died at 03:44 during the work — the state file was stale. Cause of death never determined (likely transient `_check_health_port` failure in the new single-instance code path triggering alive-but-unhealthy → stop). The URL-construction smoke would have passed regardless of liveness.

Rule: the worker's Completion Checklist line for "process X NOT killed" MUST be backed by a smoke test that does (a) `curl /health` returning 200, OR (b) `ps -p <PID>` returning a row with elapsed time advancing since the last check. Worker reports the actual HTTP code or `ps` row content, not "process was alive". This is in addition to URL-discovery smoke (which tests the discovery path), not a replacement for it.
