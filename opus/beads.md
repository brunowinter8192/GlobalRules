# Beads (Cross-Session Context)

All bead operations via the `bd` CLI. `bd` is installed with the iterative-dev plugin. Default repo: current working directory's `.beads/dolt`.

**Commands:**

| Operation | CLI |
|---|---|
| List open beads | `bd list -s open` |
| List closed beads | `bd list -s closed` |
| Show bead + comments | `bd show <id> && echo "--- COMMENTS ---" && bd comments <id>` |
| Create bead | `bd --repo <project_path> create --title "<title>" --type task --description "<desc>"` |
| Knowledge bead | add `--labels "knowledge"` to create |
| Add comment | `bd comments add <id> "<text>"` |
| Close bead | `bd close <id> --reason="<reason>"` |

**Cross-project access:** All commands except `create` accept `--db <path>/.beads/dolt`.

**Explicit-ID Rule (close, comment):** ALWAYS use the literal bead ID string in `bd close` and `bd comments add`. NEVER pipe-extract from `bd list` output via `head | awk | cut` — `bd list` is alphabetical by ID prefix, not chronological. If you need to find an ID, `bd list | grep <unique-substring>` and copy the ID literally.

## Task Management Hierarchy

- **Beads** (`.beads/`) — Cross-session entry-points (days/weeks/months)
- **Plan-File** (`.claude/plans/`) — Within a session (hours)

## What a Bead IS

A Bead is a **lean entry-point**. It tells **what the topic is** and **which sources reference it** — nothing more. The actual content (architectural decisions, discussion trail, iteration history) lives elsewhere:

- `decisions/<area>.md` — current architectural state (IST/Evidenz/SOLL)
- `decisions/OldThemes/<topic>/` (subfolder) or `decisions/OldThemes/<topic>.md` (single file) — discussion trail, iteration history, archived themes
- `<package>/DOCS.md` — module map (LOC, Called-by, Calls-out, State, Gotchas)
- RAG `<Project>_reference` collection — external papers / GitHub / Reddit
- Plan-File at `.claude/plans/<topic>.md` — current iteration's working notes (transient)

The Bead points; the sources hold the substance. RAG-search on `<Project>-features` (OldThemes), `<Project>-meta` (decisions/DOCS/CLAUDE/sources), `<Project>_reference` (papers) is the resume mechanism. See `~/.claude/shared-rules/global/tool-use.md` § "RAG-features: Bead Resume".

## Bead Format

```
Title: <Feature/Bug/Task name>
Type: task

What it is:
[2-3 sentences — goal + scope. No iteration history. No decision rationale.]

Sources referencing this topic:
- decisions: <file paths if any>
- DOCS: <DOCS.md paths if any>
- OldThemes: <subfolder or file paths if any>
- <Project>_reference: <document names if any>
- Plan-File: <path if currently active>

Resume: rag-cli search_hybrid "<query>" <Project>-meta | <Project>-features [--document "%filter%"]
```

Source paths relative to project root. The Source-Inventory is a snapshot at the moment of writing — Recap is responsible for keeping it current.

## When to Create a Bead

Beads exist to keep cross-session context alive — not to log every task in a session. Mid-session beadification fires for three narrow triggers.

**Mid-session bead triggers:**

1. **Explicitly deferred work** — user explicitly says "later", "next session", "not now". Bead captures the deferred item before context evaporates.
2. **Blocker for current work** — current task can't continue until X is fixed. Create a Bead for the blocker, work it, resume the original. If the original is still incomplete, it gets its own Bead too.
3. **Cross-session work in progress, on explicit request** — work that needs to survive the session (worker with pending verification, multi-stage investigation that won't finish today). Usually surfaces in Recap via EMPTY PLATE; only beadify mid-session if the user explicitly asks.

**NOT a bead trigger:**

- Completed one-shot work — the commit is the record.
- Worker dispatched in the same breath — the worker IS the execution.
- Things the user mentions that get done immediately — the doing is the answer.

**Recap (EMPTY PLATE):** anything still open at session end without a mid-session bead gets beadified during Recap.

**Workflow when a bead exists:**
- Default priority: finish the current bead before picking up a new one.
- Subtasks live as comments inside the bead (lean state changes only — see "Comments" below).
- **Proactive close after live-verify.** When a bead's code is merged AND live-verify shows the new behavior is working as intended, close the bead in the same flow — do NOT wait for the user to ask.

**Rule in short:** Bead = explicitly deferred OR blocker OR explicit cross-session request. Done one-shot work, in-flight workers, and same-flow tasks need no bead.

## Bead Lifecycle

- **Open**: Bead exists with current Source-Inventory at the moment of creation.
- **Active phase**: substantial work happens — Plan-Files, Worker-Outputs, code edits, discussions. Comments stay LEAN (state changes only, not narrative).
- **Recap (session end)**: writes/extends the persistent prosa files (`OldThemes/`, `decisions/`, DOCS.md) for what happened this session. Then updates the Bead's Source-Inventory if new files came into existence — via `bd comments add` (no edit-description in `bd`). See `~/.claude/shared-rules/opus/workers-3.md` § Recap.
- **Close**: `bd close <id> --reason="<one-line reason of what was achieved>"`. NO write-prosa-on-close — Recap handles persistence. NO long close-comment.

The Bead lives only as long as work is open. Once closed, the persistent prosa files stay — that is the long-term memory. Follow-up tasks pick up the thread by RAG-searching `<Project>-features` for the topic name, even years later.

## Resume Pattern

When picking up an open Bead in a new session:

1. Read the Bead (title, what-it-is, source-inventory, comments)
2. RAG-search for context:
   - `rag-cli search_hybrid "<topic>" <Project>-features [--document "%feature%"]` — discussion trail / iteration history
   - `rag-cli search_hybrid "<topic>" <Project>-meta` — current architectural state
   - `rag-cli search_hybrid "<topic>" <Project>_reference` — external papers / sources
3. Optional: targeted `rag-cli read_document` to expand a hit when the chunk doesn't carry enough.

The Bead does not contain narrative. The sources do.

## Comments — Lean State Only

Bead comments are short status changes. ONE line per comment.

Allowed:
- "Phase A done — sync-multi"
- "blocked on X — needs Y"
- "merged on dev, awaiting verification"
- "Source-Inventory updated: + decisions/eval01_methodology.md, + OldThemes/eval/decisions.md"

Forbidden:
- STAND blocks (DONE/OPEN/NEW/DROPPED/APPROACH)
- Long iteration narratives
- Decision rationale ("we chose X over Y because...")
- Multi-paragraph status reports

Iteration narrative belongs in `OldThemes/<topic>/` prosa, written by Recap. Decision rationale belongs in `decisions/<area>.md`.

## Bead-Close

`bd close <id> --reason="<one-line reason>"` — that's it.

No verification of prosa-state at close (Recap is responsible for persistence). No long close-comment summarizing the journey — the OldThemes prosa is that summary.

If a Bead defines a specific verification test that has not been run yet → Bead stays open, run the test, then close.

## Path Rule

ALL paths in Beads relative to PROJECT ROOT. Name each repo when multiple are affected.
