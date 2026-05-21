# Beads (Cross-Session Context)

All bead operations via the `bd` CLI. `bd` is installed with the iterative-dev plugin. Default repo: current working directory's `.beads/dolt`.

**Commands:**

| Operation | CLI |
|---|---|
| List open beads | `bd list -s open` |
| List closed beads | `bd list -s closed` |
| Show bead + comments | `bd show <id> && echo "--- COMMENTS ---" && bd comments <id>` |
| Create bead | `bd --repo <project_path> create --title "<title>" --type task --description "<desc>"` |
| Add comment | `bd comments add <id> "<text>"` |
| Close bead | `bd close <id> --reason="<reason>"` |

**Cross-project access:** All commands except `create` accept `--db <path>/.beads/dolt`.

**Finding IDs:** `bd list | grep <unique-substring>` — `bd list` is alphabetical by ID prefix, not chronological, so passing literal IDs by hand-copy is the safe pattern.

## Task Management Hierarchy

- **Beads** (`.beads/`) — Cross-session entry-points (days/weeks/months)
- **Plan-File** (`.claude/plans/`) — Within a session (hours)

## What a Bead IS

A Bead is a **lean entry-point**: topic + sources that reference it. Content lives elsewhere:

- `decisions/<area>.md` — current architectural state (IST/Evidenz/SOLL)
- `decisions/OldThemes/<topic>/` or `decisions/OldThemes/<topic>.md` — discussion trail, iteration history
- `<package>/DOCS.md` — module map
- RAG `<Project>_reference` collection — external papers / GitHub / Reddit
- Plan-File at `.claude/plans/<topic>.md` — transient session notes

Resume mechanism: RAG-search on `<Project>-features` (OldThemes), `<Project>-meta` (decisions/DOCS/CLAUDE/sources), `<Project>_reference` (papers).
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

**Mid-session bead triggers** (three narrow):

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

Examples:
- "Phase A done — sync-multi"
- "blocked on X — needs Y"
- "merged on dev, awaiting verification"
- "Source-Inventory updated: + decisions/eval01_methodology.md, + OldThemes/eval/decisions.md"

Iteration narrative goes to `OldThemes/<topic>/` prosa via Recap. Decision rationale lives in `decisions/<area>.md`.

## Bead-Close

`bd close <id> --reason="<one-line reason>"` — that's it.

No verification of prosa-state at close (Recap is responsible for persistence). No long close-comment summarizing the journey — the OldThemes prosa is that summary.

If a Bead defines a specific verification test that has not been run yet → Bead stays open, run the test, then close.

## Path Rule

ALL paths in Beads relative to PROJECT ROOT. Name each repo when multiple are affected.
