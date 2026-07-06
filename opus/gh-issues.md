# GitHub Issues (Cross-Session Context)

`<owner>` and `<repo>` come from the project's git remote — `git remote get-url origin` returns `github.com:<owner>/<repo>.git`. Derive them; don't hardcode.

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

## What an Issue IS

An Issue is a **lean entry-point**: topic + sources that reference it. Content lives elsewhere:

- source code — the current architectural state (read the code; there is no doc mirror)
- `process-docs/<area>/` — process history: investigation, measurements, iteration, the reasoning behind chosen code values (write-once entries)
- `<package>/DOCS.md` — module map
- RAG `<Project>-reference` collection — external sources (vendor docs, papers, GitHub, Reddit, repos)

Resume mechanism: RAG-search on `<Project>-docs` (DOCS/CLAUDE/process-docs) + reading the code, `<Project>-reference` (external sources).

Issues are created at exactly two points: when the user asks mid-session, or at Recap for whatever is still open at session end. There is no autonomous mid-session issue-keeping between those.

## Issue Format

The issue body carries the entry-point. Title = the feature/bug/task name; body:

```
What it is:
[2-3 sentences — goal + scope. No iteration history. No decision rationale.]

Sources referencing this topic:
- code: <key src/ paths if any>
- DOCS: <DOCS.md paths if any>
- process-docs: <subfolder or file paths if any>
- <Project>-reference: <document names if any>

Resume: RAG search "<query>" on <Project>-docs
```

Source paths relative to project root. The Source-Inventory is a snapshot at the moment of writing — Recap is responsible for keeping it current.

## Resume Pattern

When picking up an open issue in a new session:

1. Read the issue: `gh-cli get_issue <owner> <repo> <number>` — the body carries the Source-Inventory (no comments to read)
2. RAG search for context:
   - `<Project>-docs` — current state + discussion trail / iteration history
   - `<Project>-reference` — external sources (vendor docs, papers, repos)

The issue does not contain narrative. The sources do.

## Issue-Close

`gh-cli update_issue <owner> <repo> <number> --state closed` — that's it.

Close proactively: when the issue's code is merged AND live-verify shows the new behavior works as intended, close it in the same flow — don't wait for the user to ask.

No verification of prosa-state at close (Recap is responsible for persistence). The process-docs prosa is the journey summary — nothing is posted to the issue.

If an issue defines a specific verification test that has not been run yet → issue stays open, run the test, then close.
