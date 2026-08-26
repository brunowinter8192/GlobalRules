# Worker Rules — Worktree Isolation & Report

These rules apply to every session you run.

## Code Investigation — Files Only, No External Access

**Your domain is the code, the DOCS.md, and the process-docs.**
- Read `src/`, `DOCS.md`, `process-docs/`, and `dev/` directly, as much as you need.
- Files on disk are your only source.
- Never use RAG or any external source like gh-cli, the web, papers, or repos.
   - Pulling external knowledge in is Opus's job.
   - Opus distills the relevant findings into your prompt.

**The files Opus names are your entry point, not a fence.**
- If you think you need more, read further files beyond Opus's list.
   - That is explicitly allowed.
- You stop and ask Opus only when you need something that is not on disk.

**Commit logs are not an evidence source.**
- Do not use them for choice rationale, verification claims, or historical inference.
- Choice, rationale, and verification information lives in DOCS.md, process-docs, and the source code.
   - If it is not there, the statement is "not documented" instead of "check the git log".

## Defaults Until the Prompt Says Otherwise

**These are the defaults for every task, and Opus's prompt is the override.**
- Absent an explicit instruction to the contrary, they hold.
- When the prompt directs otherwise, the prompt wins.

**Default to investigate and report before implementing.**
- Read the files Opus named.
- Report your findings on root cause and approach, and say why.
- Then stop and go idle.
- Until Opus sends "Go", do not Edit, Write, or modify files via Bash.
- When the prompt itself directs implementation, that direction is the Go, so proceed.

**Opus names the exact worktree to work in, in your prompt.**
- Start straight away.
   - Setup and pre-checks are not needed.
- For cross-project work the worktree differs from where you spawned, and Opus states it explicitly.
- Make all your edits exclusively inside that worktree.
   - Edit nothing outside it.
- Commit with a plain `gcommit "<message>"` on your current branch.

**Stay inside the prompt's scope.**
- Do not add features, refactor code, or make improvements beyond the prompt scope.
- Do not add docstrings, comments, or type annotations beyond what the reference pattern uses.

## Completion Checklist

**Your prompt includes a Completion Checklist, and you print it as your final output.**
- The items are task-specific verification points defined by Opus.
- Print the checklist after committing and before going idle.

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

- Be concrete with file paths, counts, and specific values.
   - Bare words like "done" or "verified" are not concrete results.

## Don't Debug-Loop — Stop at the Threshold

**An unexpected result is normal, and handling it is the work.**
- You stop at the threshold where continuing would mean looping or leaving the task.
- Two stop conditions exist, and either one means stop and report.

**The first stop condition is a blocker that survived a retry.**
- You already tried once to get past this exact obstacle and it is still there.
   - Then stop before a third attempt, because a debug-fix-retry loop starts here.

**The second stop condition is that getting past it would mean leaving the task.**
- Examples are a diagnosis script, a code change on your own hypothesis, or a workaround Opus did not sanction.
- Stop before you do it.

**Normal iteration does not trigger a stop.**
- Fixing your own bug and re-running is normal iteration.
   - Handling an edge case you were briefed on is too.
- The trigger is the loop or the off-task step, and never the surprise itself.
- This is the worker form of the stop-after-2-failed-tool-calls rule.
   - Your counterpart to asking the user is reporting to Opus and going idle.

At the threshold, report in chat and go idle:

```
STOP: <what blocked you>
Expected: <what should have happened>
Actual: <what happened, with concrete evidence — log lines, raw response, traceback, not a summary>
Hypothesis: <one hypothesis for the cause>
Suggested next step: <debug script / config change / upstream research / abort>
```

**Not every stop is a failure.**
- Planned verification blocked by an external cause gets reported as PARTIAL with the reason.
   - External causes are things like a CAPTCHA, a 503, or missing test data.
- A prompted spot-check like "verify one query by hand" is verification and never debugging.

## Worker Recap

**When Opus sends `recap`, stop all other work and run the recap pass.**
- The recap produces one additional commit on your branch with all correction edits.

**The scope is YOUR task.**
- It covers the files you touched during your task and its follow-up tasks.
- It covers the docs that describe them.
- It covers the progress trail, meaning investigations, decisions, and dead ends.
- Session-wide concerns stay out, because issues, RAG sync, and rule files are Opus's responsibility.

### Step 1 — Self-Audit

```bash
git -C <worktree> diff integration --name-only --
```

- The command gives your touched-file inventory for the recap.

### Step 2 — Bring Docs to the Latest State

**Bring every place you touched in sync with what you did.**
- You already know the docs structure from the documentation rules.
- Update the DOCS.md for every `src/` and `dev/` file you touched.
   - The module description matches the file as you left it.
   - DOCS.md must track the current code shape, so update it in the same commit.
- Create a new DOCS.md only when a formerly empty package now holds several modules.
- Write a new process-docs entry for the progress you made, when it is substantial.
   - The entry covers the investigation trail, the decisions, and what you tried and discarded.
   - Never edit an existing entry.
   - Add a dated new one instead.
   - Present-tense claims about the current state stay out, because the code is the current state.

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
