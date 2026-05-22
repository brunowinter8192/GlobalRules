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
- **Active phase**: substantial work happens — Plan-Files, Worker-Outputs, code edits, discussions, investigation findings. **All narrative — including findings, status updates, blockers, progress — is written to `decisions/OldThemes/<topic>/` IMMEDIATELY when it emerges** (not deferred to Recap). Bead comments are EXCLUSIVELY Source-Inventory updates (pointers to the OldThemes/decisions/DOCS files that hold the substance).
- **Recap (session end)**: SAFETY NET for unwritten prosa — captures anything not already in `OldThemes/`/`decisions/`/DOCS.md, fixes drift, updates Bead Source-Inventory with files created this session. Recap is NOT the default workflow for narrative capture; if findings keep landing only at Recap, it means the active-phase rule is being violated. See `~/.claude/shared-rules/opus/workers-3.md` § Recap.
- **Close**: `bd close <id> --reason="<one-line reason of what was achieved>"`. NO write-prosa-on-close — Recap handles any persistence gap. NO long close-comment.

## Resume Pattern

When picking up an open Bead in a new session:

1. Read the Bead (title, what-it-is, source-inventory, comments)
2. RAG-search for context:
   - `rag-cli search_hybrid "<topic>" <Project>-features [--document "%feature%"]` — discussion trail / iteration history
   - `rag-cli search_hybrid "<topic>" <Project>-meta` — current architectural state
   - `rag-cli search_hybrid "<topic>" <Project>_reference` — external papers / sources
3. Optional: targeted `rag-cli read_document` to expand a hit when the chunk doesn't carry enough.

The Bead does not contain narrative. The sources do.

## Comments — Source-Inventory Pointers ONLY (HARD RULE)

Bead comments serve EXACTLY ONE purpose: pointing at where the substance lives. The bead is structure; the OldThemes/decisions/DOCS files are content.

**ONLY ALLOWED comment shape:**

- Source-Inventory updates: `"Source-Inventory updated: + decisions/OldThemes/<topic>/<file>.md"` / `"+ decisions/<step>.md"` / `"+ <package>/DOCS.md"`

Multiple additions in one comment are fine if they land in the same write cycle: `"Source-Inventory updated: + OldThemes/<topic>/A1.md, + decisions/pipe05.md"`.

**FORBIDDEN as bead comments (always belong in OldThemes prosa):**

- State transitions ("Phase A done", "merged on dev", "blocked on X", "awaiting verification") — even one-liners. Status is captured by which OldThemes files have been written/extended.
- Investigation findings, hypotheses, evidence comparisons
- Repro notes, screenshot timestamps, test inputs/outputs
- Anything that is not literally `Source-Inventory updated: + <path>`

**Why state-transitions are forbidden too:** they're narrative in mini-form. "Phase A done" duplicates information the OldThemes file already carries (or should carry — if it doesn't, fix the OldThemes file, don't comment the bead). The bead's purpose is to be a stable cross-session entry-point with a current Source-Inventory — not a live status feed.

**Workflow for any session activity touching a bead:**

1. Investigation finding emerges OR status changes (phase complete, blocker found, merge done, verification pending) → write/extend the relevant `decisions/OldThemes/<topic>/<file>.md` IMMEDIATELY.
2. If the write produces a NEW file: add ONE comment `Source-Inventory updated: + <new_path>`.
3. If the write extends an EXISTING file already in the Source-Inventory: no comment needed (the Source-Inventory pointer is unchanged).
4. Continue working.

**Retroactive cleanup is NOT required.** Beads polluted by narrative comments from prior sessions stay as-is — comment history is append-only. The OldThemes file becomes the canonical source once it exists; legacy bead comments are noise readers should ignore in favor of the Source-Inventory link.

Narrative of ANY shape → `decisions/OldThemes/<topic>/`. Decision rationale → `decisions/<area>.md`. Bead comments → Source-Inventory pointers only.

## Bead-Close

`bd close <id> --reason="<one-line reason>"` — that's it.

No verification of prosa-state at close (Recap is responsible for persistence). No long close-comment summarizing the journey — the OldThemes prosa is that summary.

If a Bead defines a specific verification test that has not been run yet → Bead stays open, run the test, then close.

## Path Rule

ALL paths in Beads relative to PROJECT ROOT. Name each repo when multiple are affected.
