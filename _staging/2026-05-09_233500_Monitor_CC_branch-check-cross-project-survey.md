# 2026-05-09 — Monitor_CC: Branch-Check + Cross-Project Survey

## ~/.claude/shared-rules/opus/workers-1.md → Dev-Branch Workflow section

The current "Branch-State-Check when switching to existing dev (MANDATORY)" only triggers WHEN already switching. Real failure mode observed in session 2026-05-09: session started on `main`, never executed `git checkout dev`, worker-cli detected current branch=main and merged the worker's branch directly into main (bypassing the dev → main review step).

Proposed addition as a new top-level item BEFORE "Branch-State-Check when switching":

**Session-Start Branch-Check (FIRST tool-call after Session Start):**

1. Before any worker dispatch (Phase 2 of PLAN), Opus runs `git -C <project_root> branch --show-current`.
2. If output is NOT `dev`: switch via `git checkout dev` (or `git checkout -b dev` if absent), then proceed with the existing Branch-State-Check (`git log dev..main --oneline`).
3. Only after dev is current AND not behind main → worker dispatch.

Rationale: worker-cli's merge target = current branch at merge-time. Without explicit session-start dev-checkout, merges land on whichever branch was active at session-start (default = main). Side-effect on session 2026-05-09: gpu-autostop landed on main; required manual `git checkout dev && git merge --ff-only main && git checkout main` to re-align dev for future workers in the same session.


## ~/.claude/shared-rules/opus/workers-1.md → Worker Project Scope section (new sub-section)

Add after the "Cross-project edits Opus does directly" rule:

**Pre-PLAN Cross-Project State Survey (MANDATORY when work touches secondary repo):**

When PLAN identifies that work will touch a secondary repo (cross-project, e.g. Monitor_CC session needs RAG edits), Opus MUST do a pre-PLAN state survey of that repo BEFORE scoping the worker:

- `git -C <secondary_repo> branch --show-current` + `git -C <secondary_repo> branch -a` — current branch + branch list
- Diff between branches that may be merge targets — `git -C <secondary_repo> diff main..dev -- <expected_touched_files>` or similar
- Identify divergent constants, paths, defaults that the work might unknowingly preserve, invert, or mis-target

Failure mode observed session 2026-05-09: gpu-autostop worker received instruction "use ~/.rag-locks for the timestamp path (main-branch production location)" but worked in a `dev` worktree where `TIMESTAMP_DIR = Path("/tmp")` was hardcoded. Worker correctly preserved dev's value per Opus's instruction "Do NOT touch the dev-branch defaults". Live-verify after merge surfaced the mismatch — pane reads from `~/.rag-locks` but watchdog (running dev's code) wrote to `/tmp`.

The discrepancy was visible in the worker's Phase A findings ("two critical discrepancies vs the prompt") and Opus told worker to leave dev unchanged. Pre-PLAN survey would have surfaced this earlier and resulted in either:

(a) include cross-branch alignment in worker scope from the start (single Phase B fixes both files),
(b) explicitly defer with risk-acknowledgment in PLAN ("we accept dev/main divergence; live-verify will require post-merge fix").

Either outcome is fine — what's not fine is: instructing the worker to ignore the divergence and getting bitten at live-verify.
