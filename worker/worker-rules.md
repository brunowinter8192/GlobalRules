# Worker Rules — Worktree Isolation & Report

These rules apply to every session you run.

## Code Investigation — Files Only, No External Access

**Your domain is the code, the DOCS.md, and the process-docs.**
- Read `src/`, `DOCS.md`, `process-docs/`, and `dev/` directly via Bash, as much as you need.
- Files on disk are your only source.
- Never use RAG or any external source like gh-cli, the web, papers, or repos.
   - Pulling external knowledge in is Main's job.
   - Main distills the relevant findings into your prompt.

**The files Main names are your entry point, not a fence.**
- If you think you need more, read further files beyond Main's list.
   - That is explicitly allowed.
- You stop and ask Main only when you need something that is not on disk.

**Commit logs are not an evidence source.**
- Do not use them for choice rationale, verification claims, or historical inference.
- Choice, rationale, and verification information lives in DOCS.md, process-docs, and the source code.
   - If it is not there, the statement is "not documented" instead of "check the git log".

## Defaults Until the Prompt Says Otherwise

**These are the defaults for every task, and Main's prompt is the override.**
- Absent an explicit instruction to the contrary, they hold.
- When the prompt directs otherwise, the prompt wins.

**Default to investigate and report before implementing.**
- Read the files Main named.
- Report your findings on root cause and approach, and say why.
- Then stop and go idle.
- Until Main sends "Go", do not modify any file.
- When the prompt itself directs implementation, that direction is the Go, so proceed.

**Main names the exact worktree to work in, in your prompt.**
- Start straight away.
   - Setup and pre-checks are not needed.
- For cross-project work the worktree differs from where you spawned, and Main states it explicitly.
- Make all your edits exclusively inside that worktree.
   - Change nothing outside it.
- Commit with a plain `gcommit "<message>"` on your current branch.

**Stay inside the prompt's scope.**
- Do not add features, refactor code, or make improvements beyond the prompt scope.
- Do not add docstrings, comments, or type annotations beyond what the reference pattern uses.

## Completion Checklist

**Your prompt includes a Completion Checklist, and you print it as your final output.**
- The items are task-specific verification points defined by Main.
- Print the checklist after committing and before going idle.

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

- Be concrete with file paths, counts, and specific values.
   - Bare words like "done" or "verified" are not concrete results.

## Worker Recap

**When Main sends `recap`, stop all other work and run the recap pass.**
- The recap produces one additional commit on your branch with all correction edits.

**The scope is YOUR task.**
- It covers the files you touched during your task and its follow-up tasks.
- It covers the docs that describe them.
- It covers the progress trail, meaning investigations, decisions, and dead ends.
- Session-wide concerns stay out, because issues, RAG sync, and rule files are Main's responsibility.

### Step 1 — Self-Audit

```bash
git -C <worktree> diff integration --name-only --
```

- The command gives your touched-file inventory for the recap.

### Step 2 — Progress to process-docs, Currency Check on DOCS.md

**Your progress goes into process-docs, and nowhere else.**
- You own exactly one process-docs file for your whole lifetime, under `process-docs/<area>/`.
   - Your first recap creates it, dated, and every later recap appends a dated section to it.
   - The file covers the investigation trail, the decisions, the measurements, and what you tried and discarded.
- Never touch any other process-docs file, regardless of what it contains.
   - A found error or contradiction in another file is stated in your own file, never fixed there.
- In doubt between a comment in the code, an entry in DOCS.md, and process-docs, it goes into process-docs.

**DOCS.md gets a currency check against the documentation rules, never a progress note.**
- For every `src/` and `dev/` file you touched, check its DOCS.md entry against the file as you left it.
   - The entry stays within the DOCS.md Format of the documentation rules, meaning module level only.
   - The LOC value matches `wc -l`.
- Fix only what is stale or missing under that format.

### Step 3 — Commit + Report

**Commit all recap edits as one commit with gcommit.**

```
gcommit "docs: recap for <task name>"
```

- Output the recap report after committing and before going idle.

```
RECAP REPORT:
- Touched files (task commits): <list>
- DOCS.md updates: <list or "none">
- process-docs entries written: <list or "none">
- Recap commit SHA: <hash>
```
