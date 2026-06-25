# GitHub Issues (Cross-Session Context)

All issue operations via the `gh-cli` CLI (Python tool — handles auth via the `~/.zshrc` token and the mitmproxy transport transparently; no proxy/cert flags needed). Issues live in the project's GitHub repo `<owner>/<repo>` — owner is `brunowinter8192`, repo = project folder name (e.g. `monitor-cc`, `rag-cli`, `gh-cli`, `trading`).

**Commands:**

| Operation | CLI |
|---|---|
| List open issues | `gh-cli list_issues <owner> <repo>` (state=open is the default) |
| List closed issues | `gh-cli list_issues <owner> <repo> --state closed` |
| Read issue body | `gh-cli get_issue <owner> <repo> <number>` — body = text AFTER the `---` separator in the output |
| Create issue | `gh-cli create_issue <owner> <repo> "<title>" --body "<desc>" [--labels a,b]` |
| Update issue body (Source-Inventory) | `gh-cli update_issue <owner> <repo> <number> --body "<full updated body>"` (full-replace) |
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

Resume: RAG search "<query>" on <Project>-docs
```

Source paths relative to project root. The Source-Inventory is a snapshot at the moment of writing — Recap is responsible for keeping it current.

## When to Create an Issue

**Mid-session issue triggers** (three narrow):

1. **Explicitly deferred work** — user explicitly says "later", "next session", "not now".
2. **Blocker for current work** — current task can't continue until X is fixed. Create an issue for the blocker, work it, resume the original. If the original is still incomplete, it gets its own issue too.
3. **Cross-session work in progress, on explicit request** — work that needs to survive the session (worker with pending verification, multi-stage investigation that won't finish today). Usually surfaces in Recap via EMPTY PLATE; only create mid-session if the user explicitly asks.

**NOT an issue trigger:**

- Completed one-shot work — the commit is the record.
- Worker dispatched in the same breath — the worker IS the execution.
- Things the user mentions that get done immediately — the doing is the answer.

**Recap (EMPTY PLATE):** anything still open at session end without a mid-session issue gets an issue during Recap.

**Workflow when an issue exists:**
- Default priority: finish the current issue before picking up a new one.
- Source-Inventory updates live in the issue **body**, maintained via `update_issue --body` (read current body via `get_issue`, splice in new paths, full-replace — see "Source-Inventory — Body-Maintained" below). NO comments are used.
- **Proactive close after live-verify.** When an issue's code is merged AND live-verify shows the new behavior is working as intended, close the issue in the same flow — do NOT wait for the user to ask.

**Rule in short:** Issue = explicitly deferred OR blocker OR explicit cross-session request. Done one-shot work, in-flight workers, and same-flow tasks need no issue.

## Issue Lifecycle

- **Open**: Issue exists with current Source-Inventory at the moment of creation.
- **Active phase**: substantial work happens — Worker-Outputs, code edits, discussions, investigation findings. **All narrative — including findings, status updates, blockers, progress — is written to `decisions/OldThemes/<topic>/` IMMEDIATELY when it emerges** (not deferred to Recap). The issue **body's** Source-Inventory is updated via `update_issue` to point at the OldThemes/decisions/DOCS files that hold the substance. No comments are used.
- **Recap (session end)**: SAFETY NET for unwritten prosa — captures anything not already in `OldThemes/`/`decisions/`/DOCS.md, fixes drift, updates the Source-Inventory with files created this session. Recap is NOT the default workflow for narrative capture; if findings keep landing only at Recap, it means the active-phase rule is being violated. See `~/.claude/shared-rules/opus/workers-3.md` § Recap.
- **Close**: `gh-cli update_issue <owner> <repo> <number> --state closed`. NO write-prosa-on-close — Recap handles any persistence gap.

## Resume Pattern

When picking up an open issue in a new session:

1. Read the issue: `gh-cli get_issue <owner> <repo> <number>` — the body carries the Source-Inventory (no comments to read)
2. RAG search for context:
   - `<Project>-docs` — current state + discussion trail / iteration history
   - `<Project>-reference` — external papers / sources

The issue does not contain narrative. The sources do.

## Source-Inventory — Body-Maintained (HARD RULE)

There are NO issue comments. `comment_issue` does not exist. The Source-Inventory lives in the issue **body** and is the ONLY part that changes after creation. The issue body is structure (the entry-point); the OldThemes/decisions/DOCS files are the content.

**The body is a 3-part document** (see Issue Format): `What it is` (stable), `Sources referencing this topic` (the live part), `Resume` (stable).

**Read-modify-write workflow** — `update_issue --body` is a FULL REPLACE, so the stable parts must be preserved:

1. `gh-cli get_issue <owner> <repo> <number>` → read the current body. The editable body is the text AFTER the `---` separator (the lines before it are display metadata: title/state/author/dates/labels/URL).
2. Keep `What it is` and `Resume` verbatim; splice the new source path(s) into the `Sources referencing this topic` list.
3. `gh-cli update_issue <owner> <repo> <number> --body "<full reconstructed body>"`.

`What it is` and `Resume` are written once at create and stay put unless the topic's scope genuinely changes. Only the Source-Inventory list grows.

**FORBIDDEN in the body (always belong in OldThemes prosa):**

- State transitions ("Phase A done", "merged on integration", "blocked on X", "awaiting verification")
- Commit SHAs / merge announcements / fix-landed phrasing
- Live-test instructions or verification steps
- Investigation findings, hypotheses, evidence comparisons
- Repro notes, screenshot timestamps, test inputs/outputs
- Anything that is not a source path

The `Sources referencing this topic` list is JUST paths. Anything that is not a source path does not belong in the issue at all — it goes to OldThemes.

**Workflow for any session activity touching an issue:**

1. Investigation finding emerges OR status changes (phase complete, blocker found, merge done, verification pending) → write/extend the relevant `decisions/OldThemes/<topic>/<file>.md` IMMEDIATELY.
2. If the write produces a NEW file (OldThemes/decisions/DOCS) → read-modify-write the issue body to add the path to the Source-Inventory.
3. If it only extends a file already listed → no body change needed.
4. Continue working.

Narrative of ANY shape → `decisions/OldThemes/<topic>/`. Decision rationale → `decisions/<area>.md`. Issue body → Source-Inventory paths only.

## Issue-Close

`gh-cli update_issue <owner> <repo> <number> --state closed` — that's it.

No verification of prosa-state at close (Recap is responsible for persistence). The OldThemes prosa is the journey summary — nothing is posted to the issue.

If an issue defines a specific verification test that has not been run yet → issue stays open, run the test, then close.

## Path Rule

ALL paths in issue bodies relative to PROJECT ROOT. Name each repo when multiple are affected.
