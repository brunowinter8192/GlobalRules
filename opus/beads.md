# Beads (Cross-Session Context)

All bead operations via the `bd` CLI. `bd` is installed with the iterative-dev plugin. Default repo: current working directory's `.beads/dolt`.

**Commands:**

| Operation | CLI | Notes |
|---|---|---|
| List open beads | `bd list -s open` | Run at every session start |
| List closed beads | `bd list -s closed` | |
| Show bead + comments | `bd show <id> && echo "--- COMMENTS ---" && bd comments <id>` | Two calls merged |
| Create bead | `bd --repo <project_path> create --title "<title>" --type task --description "<desc>"` | `create` uses `--repo`. Type is always `task`. |
| Create knowledge bead | add `--labels "knowledge"` to create | |
| Add comment | `bd comments add <id> "<text>"` | |
| Close bead | `bd close <id> --reason="<reason>"` | Reason must explain WHAT was done |

**Cross-project access:** All commands except `create` accept `--db <path>/.beads/dolt` (e.g. `bd --db /path/to/project/.beads/dolt list -s open`).

**Explicit-ID Rule (close, comment):** ALWAYS use the literal bead ID string in `bd close` and `bd comments add` calls. NEVER pipe-extract from `bd list` output via `head | awk | cut`. `bd list` output is alphabetical by ID prefix, NOT chronological — extracted ID will silently target the wrong bead. If you just created bead X in the previous tool call, use X directly. If you need to find an ID from a list, `bd list | grep <unique-substring>` and read the ID manually before using it.
- Concrete failure (2026-04-23): RECAP closed `blank-yy4` (just created + merged) via `bd close blank-$(bd list -s open | head -1 | awk '{print $2}' | cut -d- -f2)`. Extracted `blank-8am` (first alphabetical) instead, closed with wrong reason. Reopen + correct close required.

## Task Management Hierarchy

- **Beads** (`.beads/`) — Cross-session (days/weeks/months)
- **Plan-File** (`.claude/plans/`) — Within a session (hours)
- **TodoWrite** — Within an iteration (minutes)

## When to Create a Bead

Beads exist to keep cross-session context alive — not to log everything that happens in a session. Mid-session beadification only fires for three narrow triggers.

**Mid-session bead triggers:**

1. **Explicitly deferred work** — user mentions a task and we explicitly decide NOT to do it now ("machen wir später", "nächste Session", "nicht jetzt"). The bead captures the deferred item at the moment of deferral, before context evaporates.
2. **Blocker for current work** — current task can't continue until something else is fixed first. Create a bead for the blocker, work the blocker, resume the original task. If the original task is still incomplete, it gets its own bead too.
3. **Cross-session work in progress, on explicit request** — work that needs to survive the session (worker with pending verification, multi-stage investigation that won't finish today). Usually surfaces at Recap via EMPTY PLATE; only beadify mid-session if the user explicitly asks.

**NOT a bead trigger:**

- **Completed one-shot work** — task starts and ends in the same response or short flow. The commit is the record. No bead.
- **Worker dispatched in the same breath** — the worker IS the execution. If it spans the session, Recap catches it. No bead at dispatch time.
- **Things the user mentions that get done immediately** — the doing is the answer.

**Recap (EMPTY PLATE):** anything still open at session end that didn't get a mid-session bead gets beadified during Recap. That's the safety net for in-progress work which crossed the session boundary without an explicit mid-session trigger.

**Workflow when a bead exists:**

- Default priority: finish the current bead before picking up a new one.
- Subtasks live as comments inside the bead.
- Each subtask done → add a comment with what was done + decisions + findings.
- Deliverables done, testing pending → comment, run tests, close on pass.
- **Proactive close after live-verify.** When a bead's code is merged AND live-verify shows the new behavior is working as intended, close the bead in the same flow — do NOT wait for the user to ask "warum so viele offen?". After every `worker_merge`: ask "is this bead now closeable?". If live-verify is "user has already used the new code without complaint" or "monitor restart shows the feature working" → close. Concrete failure (2026-04-28): session accumulated 6+ open beads (q0t, pf3, 1fo) where code was merged and verified in flight. User: "nach meinem verständnis dürften jetzt nur noch 2 beads offen sein". One exchange wasted on cleanup that should have happened proactively.

**Rule in short:** Bead = explicitly deferred OR blocker OR explicit cross-session request. Done one-shot work, in-flight workers, and same-flow tasks need no bead.

## Content Requirements

Every bead MUST be understandable WITHOUT the original session context.

**On Create:** Description answers: What? Why? Where? (repos, files affected)
- Bad title: "Fix bug" — Good: "Fix NaN in search scores"

**Comments:** Capture SESSION NARRATIVE, not just outcomes.
- Include: Decisions + WHY, Alternatives rejected, Research findings, Reasoning gaps
- Bad: "Implemented X, Y, Z." — Good: "Chose X over Y because Z. Rejected A (redundant with B)."

**Progress Tracking — Session-end STAND block (MANDATORY):**
```
STAND:
- DONE: Point X, Y
- OPEN: Point Z, W
- NEW: Point V (added because ...)
- DROPPED: Point U (reason)
- APPROACH: How the work was done (workflow, worker patterns, what worked). A fresh Claude must be able to replicate the approach without guessing.
```

**On Close:** Reason explains WHAT was done, not just "done".

**Bead-Close Verification (MANDATORY):** When a Bead defines a specific verification test (e.g. "spawn 2 workers sequentially, check CR"), that test MUST be EXECUTED before closing. Plausible explanations are NOT a substitute for running the defined test. If the test cannot be run in the current session → Bead stays open.

Concrete failure (2026-04-16): Bead d84 required "2 worker sequential spawn, check 2nd worker CR". Opus closed it with "by design, kein Bug" without running the test. User had to reopen.

**Path Rule:** ALL paths relative to PROJECT ROOT. Name each repo when multiple are affected.
