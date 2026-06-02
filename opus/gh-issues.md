# GitHub Issues (Cross-Session Context)

All issue operations via the `gh-cli` CLI (Python tool — handles auth via the `~/.zshrc` token and the mitmproxy transport transparently; no proxy/cert flags needed). Issues live in the project's GitHub repo `<owner>/<repo>` — owner is `brunowinter8192`, repo = project folder name (e.g. `monitor-cc`, `rag-cli`, `gh-cli`, `trading`).

**Commands:**

| Operation | CLI |
|---|---|
| List open issues | `gh-cli list_issues <owner> <repo>` (state=open is the default) |
| List closed issues | `gh-cli list_issues <owner> <repo> --state closed` |
| Show issue + comments | `gh-cli get_issue <owner> <repo> <number> && gh-cli get_issue_comments <owner> <repo> <number>` |
| Create issue | `gh-cli create_issue <owner> <repo> "<title>" --body "<desc>" [--labels a,b]` |
| Add comment | `gh-cli comment_issue <owner> <repo> <number> "<text>"` |
| Close issue | `gh-cli update_issue <owner> <repo> <number> --state closed` |
| Reopen issue | `gh-cli update_issue <owner> <repo> <number> --state open` |

**Default = open only:** `list_issues` shows OPEN issues by default; closed appear only with `--state closed` (or `--state all`). Pull requests are filtered out.

**Finding the number:** `gh-cli list_issues <owner> <repo>` lists open issues one per line (`#N [OPEN] title`); match by title and pass the stable `<number>`.

## Task Management

**GitHub Issues** — the only task-management primitive. Cross-session entry-points (days/weeks/months). One repo per project.

## What an Issue IS

An Issue is a **lean entry-point**: topic + sources that reference it. Content lives elsewhere:

- `decisions/<area>.md` — current architectural state (IST/Evidenz/SOLL)
- `decisions/OldThemes/<topic>/` or `decisions/OldThemes/<topic>.md` — discussion trail, iteration history
- `<package>/DOCS.md` — module map
- RAG `<Project>-reference` collection — external papers / GitHub / Reddit

Resume mechanism: RAG-search on `<Project>-docs` (decisions/DOCS/CLAUDE/OldThemes), `<Project>-reference` (papers).

## Issue Format

The issue body carries the entry-point. Title = the feature/bug/task name; body:

```
What it is:
[2-3 sentences — goal + scope. No iteration history. No decision rationale.]

Sources referencing this topic:
- decisions: <file paths if any>
- DOCS: <DOCS.md paths if any>
- OldThemes: <subfolder or file paths if any>
- <Project>-reference: <document names if any>

Resume: rag-cli search_hybrid "<query>" <Project>-docs [--document "%filter%"]
```

Source paths relative to project root. The Source-Inventory is a snapshot at the moment of writing — Recap is responsible for keeping it current.

## When to Create an Issue

**Mid-session issue triggers** (three narrow):

1. **Explicitly deferred work** — user explicitly says "later", "next session", "not now". The issue captures the deferred item before context evaporates.
2. **Blocker for current work** — current task can't continue until X is fixed. Create an issue for the blocker, work it, resume the original. If the original is still incomplete, it gets its own issue too.
3. **Cross-session work in progress, on explicit request** — work that needs to survive the session (worker with pending verification, multi-stage investigation that won't finish today). Usually surfaces in Recap via EMPTY PLATE; only create mid-session if the user explicitly asks.

**NOT an issue trigger:**

- Completed one-shot work — the commit is the record.
- Worker dispatched in the same breath — the worker IS the execution.
- Things the user mentions that get done immediately — the doing is the answer.

**Recap (EMPTY PLATE):** anything still open at session end without a mid-session issue gets an issue during Recap.

**Workflow when an issue exists:**
- Default priority: finish the current issue before picking up a new one.
- Subtasks live as comments inside the issue (lean state changes only — see "Comments" below).
- **Proactive close after live-verify.** When an issue's code is merged AND live-verify shows the new behavior is working as intended, close the issue in the same flow — do NOT wait for the user to ask.

**Rule in short:** Issue = explicitly deferred OR blocker OR explicit cross-session request. Done one-shot work, in-flight workers, and same-flow tasks need no issue.

## Issue Lifecycle

- **Open**: Issue exists with current Source-Inventory at the moment of creation.
- **Active phase**: substantial work happens — Worker-Outputs, code edits, discussions, investigation findings. **All narrative — including findings, status updates, blockers, progress — is written to `decisions/OldThemes/<topic>/` IMMEDIATELY when it emerges** (not deferred to Recap). Issue comments are EXCLUSIVELY Source-Inventory updates (pointers to the OldThemes/decisions/DOCS files that hold the substance).
- **Recap (session end)**: SAFETY NET for unwritten prosa — captures anything not already in `OldThemes/`/`decisions/`/DOCS.md, fixes drift, updates the Source-Inventory with files created this session. Recap is NOT the default workflow for narrative capture; if findings keep landing only at Recap, it means the active-phase rule is being violated. See `~/.claude/shared-rules/opus/workers-3.md` § Recap.
- **Close**: `gh-cli update_issue <owner> <repo> <number> --state closed`. NO write-prosa-on-close — Recap handles any persistence gap. NO long close-comment.

## Resume Pattern

When picking up an open issue in a new session:

1. Read the issue: `gh-cli get_issue <owner> <repo> <number>` (+ `get_issue_comments` for the Source-Inventory pointers)
2. RAG-search for context:
   - `rag-cli search_hybrid "<topic>" <Project>-docs [--document "%feature%"]` — current state + discussion trail / iteration history
   - `rag-cli search_hybrid "<topic>" <Project>-reference` — external papers / sources
3. Optional: targeted `rag-cli read_document` to expand a hit when the chunk doesn't carry enough.

The issue does not contain narrative. The sources do.

## Comments — Source-Inventory Pointers ONLY (HARD RULE)

Issue comments serve EXACTLY ONE purpose: pointing at where the substance lives. The issue is structure; the OldThemes/decisions/DOCS files are content.

**ONLY ALLOWED comment shape:**

- Source-Inventory updates: `"Source-Inventory updated: + decisions/OldThemes/<topic>/<file>.md"` / `"+ decisions/<step>.md"` / `"+ <package>/DOCS.md"`

Multiple additions in one comment are fine if they land in the same write cycle: `"Source-Inventory updated: + OldThemes/<topic>/A1.md, + decisions/pipe05.md"`.

**FORBIDDEN as issue comments (always belong in OldThemes prosa):**

- State transitions ("Phase A done", "merged on dev", "blocked on X", "awaiting verification") — even one-liners. Status is captured by which OldThemes files have been written/extended.
- **Commit SHAs / merge announcements / fix-landed phrasing** ("Fix landed dev commit dcb6296", "Merged on dev", "Worker X dispatched")
- **Live-test instructions or verification steps** ("Live-test braucht menubar restart", "Run X to verify")
- **Related-fix mentions or sidenotes** ("Plus also Y kalibriert auf Z", "Bezug zu Issue #L")
- Investigation findings, hypotheses, evidence comparisons
- Repro notes, screenshot timestamps, test inputs/outputs
- Anything that is not literally `Source-Inventory updated: + <path>`

**Why state-transitions are forbidden too:** they're narrative in mini-form. "Phase A done" duplicates information the OldThemes file already carries (or should carry — if it doesn't, fix the OldThemes file, don't comment the issue). The issue's purpose is to be a stable cross-session entry-point with a current Source-Inventory — not a live status feed.

**Pre-Post Self-Test (every `comment_issue` call):**

> "Does my comment literally start with `Source-Inventory updated: + `? If no → STOP. The substance belongs in OldThemes; the comment is the pointer."

If the comment carries ANY of: a commit-SHA, a `landed/merged/dispatched/awaiting/blocked/done/pending` word, a verification instruction, a "Plus also..." sidenote — it is a rule violation. Rewrite to lean form OR delete entirely (if no new OldThemes file was created, no comment is needed at all).

**Workflow for any session activity touching an issue:**

1. Investigation finding emerges OR status changes (phase complete, blocker found, merge done, verification pending) → write/extend the relevant `decisions/OldThemes/<topic>/<file>.md` IMMEDIATELY.
2. If the write produces a NEW file: add ONE comment `Source-Inventory updated: + <new_path>`.
3. If the write extends an EXISTING file already in the Source-Inventory: no comment needed (the Source-Inventory pointer is unchanged).
4. Continue working.

Narrative of ANY shape → `decisions/OldThemes/<topic>/`. Decision rationale → `decisions/<area>.md`. Issue comments → Source-Inventory pointers only.

## Issue-Close

`gh-cli update_issue <owner> <repo> <number> --state closed` — that's it.

No verification of prosa-state at close (Recap is responsible for persistence). No long close-comment summarizing the journey — the OldThemes prosa is that summary.

If an issue defines a specific verification test that has not been run yet → issue stays open, run the test, then close.

## Path Rule

ALL paths in issue bodies/comments relative to PROJECT ROOT. Name each repo when multiple are affected.
