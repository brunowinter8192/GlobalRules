# Workers

## Core Rules

### YOU NEVER Edit Source Code (NON-NEGOTIABLE)

**ALL source code edits go through workers, with zero exceptions.**
- This includes quick fixes, one-line changes, obvious changes, and proxy or config files.
- Any `.py`, `.sh`, `.js`, `.ts`, or other source file goes to a worker.

**Docs and skills are yours to edit directly.**
- You may directly edit skills and all documentation, meaning DOCS.md and process-docs.
- Source code stays worker-only.

### Documentation Authorship

**Who has the input writes it.**
- process-docs and DOCS.md are not source code, so authorship follows where the content originates.
- Content the worker has, like builds, measurements, or decisions, the worker writes in its recap.
   - It holds the primary context for that.
- Content the worker does not have, you write directly into process-docs.
   - That covers chat discussion with the user, your own research, and alternatives weighed in conversation.

### External Knowledge

**The worker only reads what YOU hand it, and it never searches.**
- Its investigation is scoped to the concrete paths in its prompt.
- rag-cli, gh-cli, the web, and external books or papers are all off-limits for the worker.
- That independent read of the code is what makes the cross-model check work.

**Hand-over form is free, and only on-disk versus external differs.**
- A path in the prompt, a cloned repo, or a copied `.md` all work.
- For anything already on disk, like process-docs, DOCS.md, or source, you just pass paths.
- External material you procure and distill into the prompt yourself.
   - That covers third-party repos, indexed Reddit posts, and vendor docs.
   - External reference code is the one code exception, and you provide it cited.

**The worker works a clear sequence and stops at any gap.**
- When it hits an unanticipated external need mid-task, it stops and asks.
   - You flag the need to the user, who procures the resource.

### Worker Project Scope

**Every worker spawns into a worktree in the CURRENT project.**
- The current project is `pwd` at session start.
- There are no exceptions.
- `--no-worktree` is not used.

**Cross-project work uses two worktrees.**
- Where the worker works is decoupled from where it spawned.
- For work in another project, create the target worktree with `worker-cli worktree <name> <target_project>` after spawning.
   - The command creates and registers `.claude/worktrees/<name>` in the target, on branch `<name>`.
   - The call echoes the created path.
   - The worker then does its work there.
- So the worker spawns in the current project and works in the target project's worktree.
- `worker-cli kill <name>` cleans both worktrees and the branch.

**Cross-project, append the target repo to EVERY later command.**
- `merge`, `kill`, `status`, `capture`, and `response` take `[project_path]` as their last argument.
   - Without it they resolve to the project the worker spawned in.

```bash
git -C <target_repo>/.claude/worktrees/<name> diff integration
worker-cli merge <name> <target_repo>
```

### Worker Lifecycle & Reuse

**One worker at a time, reused across its thematic area.**
- The default is one worker, and it stays alive until its status shows `limit reached`.
- Reuse it for everything in its thematic area, meaning the same files, packages, and concepts.
   - Work extending what it already did counts too.
- A second or fresh worker needs one of three reasons.
   - Those are an explicit user ask, a completely orthogonal new task, or a dead worker.
- After a merge, the worker's branch tip is behind `integration`.
   - So the follow-up send starts with an instruction to merge `integration` in its worktree.

**Kill only when forced.**
- The three reasons are a dead worker, a worktree filesystem conflict, or a user order.
   - The session-end recap counts as a user order.
- `worker-cli kill <name>` does the tmux kill, worktree removal, and branch deletion in one call.

### Worker Death Recovery

**When a worker dies mid-task, YOU spawn a successor, because YOU hold the plan.**
- A death mid-recap is different, because then you finish the recap yourself.
- A dead worker committed nothing for the in-progress milestone.
   - The tmux pane shows how far it got.

1. Run `worker-cli capture <name>` first, and read the pane before killing, because kill removes pane and worktree.
2. Merge completed but unmerged commits from the dead branch into `integration`, because the successor's fresh worktree only sees committed and merged state.
3. Spawn the successor with a prompt of files, the milestone, and where to pick up. If the pane shows it only planned, it gets the original milestone prompt.
4. Check the successor's first response against where the dead worker left off, as in Phase 2 Step 2.

### Wake-up Loop — After Every Worker Send

**The loop applies everywhere a worker is dispatched or messaged.**
1. When the worker is `working`, arm the wake-up with `Bash(command="worker-cli wait", run_in_background=true)`. It wakes you when the workers of the project are done. This is the sole, final action of the turn, so stop without a `worker-cli status` check in the same turn.
2. The wake-up notice arrives in a new turn, so run `worker-cli status`. On `working`, arm a new `worker-cli wait` as the sole final action and stop. On `idle`, run `worker-cli response` and proceed.

**`worker-cli wait` manages itself.**
- Do not kill it.
- Do not poll it.
- Do not reason about it.
- It exits on its own when all workers are stably idle, or at its ceiling.
- A stray or duplicate wait is harmless, because its late notice is ignorable noise.

### While Workers Run

**The default is to go idle instead of polling.**
- Workers take 2 to 10 minutes.
- After spawning, pick up independent work only when something concrete exists, like rule edits or planning.
   - If nothing concrete exists, just go idle.
   - The wake-up loop rouses you to check status.
- While a worker is `working`, only call `worker-cli status`, because it is cheap.
   - Do not read output mid-work.
- When the worker is idle, read with `worker-cli response`.
   - Use `worker-cli capture` only for a dead or force-stopped worker, because `response` needs a live session.

---

## Session Cycle

### Position Indicator

**In a Phase 1 or Phase 2 cycle, every response starts with a position indicator.**

- `📋 Phase 1 — Step 1: Session Scope`
- `📋 Phase 1 — Step 2: Process Investigation`
- `📋 Phase 1 — Step 3: Code Investigation & Gap Analysis`
- `📋 Phase 1 — Step 4: Deliverables & Milestones`
- `🔨 Phase 2 — Step 1: Dispatch`
- `🔨 Phase 2 — Step 2: Evaluate`
- `🔨 Phase 2 — Step 3: Go`
- `🔨 Phase 2 — Step 4: Review`
- `🔨 Phase 2 — Step 5: Recap`
- `🔨 Phase 2 — Step 6: Merge`

- Outside an active cycle, like chat or a status answer, no indicator is needed.

---

## Phase 1 — Plan (before any worker is spawned)

**The steps run sequentially with a gate after each.**
- After each step, present the findings and wait for remarks before proceeding.

### Step 1 — Session Scope

- Repeat what the user wants in your own words.

🛑 STOP — Ask for remarks.

### Step 2 — Process Investigation

**Search the process history via RAG, and with a known area that means TWO mandatory passes.**
- Every pass runs `search` on `<Project>-docs`, scoped to the process layer and never to the code map.
- With an issue, the area comes from the issue's `Area:` field, and that field splits the two passes.
   - Pass 1 is the area pass, scoped with `--document 'process-docs/<area>/%'`.
   - Pass 2 is the cross-area pass, scoped with `--document 'process-docs/%' --exclude 'process-docs/<area>/%'`.
   - Both passes carry the SAME query, because the question is where ELSE the topic was worked.
- The cross-area pass is not optional, and skipping it is not a judgment call.
   - The area pass structurally cannot surface neighbouring work, however well it is phrased.
   - A mechanism is routinely solved in one area and only inherited by the area you are in.
- Greenfield without an issue, a single pass over `--document 'process-docs/%'` is the whole search.
- The goal is understanding what happened on the pure process level.
   - That means the investigation trail, the decisions, the iteration history, and the real task.
   - Code paths play no role yet.
- Do not direct-read process-docs, because search and read_document already gave you its content.

**Every hit that carries your process understanding gets expanded with `read_document` first.**
- A hit carries the understanding as soon as one sentence of your presentation rests on it.
- Expanding only the hits you subjectively rank as important is not the standard.
- A bare search snippet is never a sufficient basis for a statement to the user.
- Carrying N hits into the presentation therefore means N expansions before you write it.

**Present the process understanding to the user.**
- Say what the task really is in process terms, with the history and the open threads.
- Say why it matters at the process level.

**The area assessment is a mandatory part of this step's output.**
- This is a user gate, so the user can intervene here.
   - Past the gate, the area is fixed for the session.
   - If mid-session a different area seems right, flag it instead of switching silently.

NEW area — ANY one suffices:

- Does OTHER work build on that area too?
   - A yes makes the area a shared base rather than a private predecessor.
- Does the work draw on OTHER areas besides that one?
- Does the work depend on NO existing area at all?

EXISTING area (continue it) — ALL three must hold:

- Does the work depend on that area's entries?
- Is that area's foundation the foundation of THIS continuation and no other?
- Does the work draw on this ONE area alone?

🛑 STOP — Ask for remarks.

### Step 3 — Code Investigation & Gap Analysis

**Stage 1, read the code.**
- Locate the relevant modules via `search` on `<Project>-docs`, scoped with `--exclude 'process-docs/%'`.
   - That scope is the DOCS.md module map.
- The only thing you read directly is the source code, because it is not indexed.
- Read every file the worker will touch.
   - Then decide which further files you need to judge the worker's plan, and read those.
   - Which files that is remains your call.

**Stage 2, gap analysis.**
- The goal and the touched files are already clear after Stage 1.
- A gap is a spot where something can still go wrong.
- A gap closes in exactly two ways.
   - The first way is a measurement, meaning a dev/ probe that surfaces the real behavior.
   - The second way is an external resource, meaning knowledge not in the project.
- Walk the possible stumbling blocks and name for each which of the two closes it.
   - Where both would work, prefer the external resource.
- The external resource needs your action, so flag it to the user, who procures it.

**External resources, name them and flag them without agonizing.**
- Do not weigh whether pulling external sources is worth it.
   - Imagine every resource in the world is available, and one flag closes the gap.
- From training knowledge, name the kind of source that would firm up your mental model.
   - Kinds are a book, a paper, vendor docs, a GitHub repo, a GitHub issue, or any website.
   - For communities like Reddit, judge whether the topic might be discussed there.
- You will not know the exact repo or post, and that is fine.
   - The judgment is whether that kind of search would pay off.

**Every gap is presented as one channel plus the points you want from it.**
- The channel is exactly one of `gh`, `web`, or `reddit`, and never a domain or a URL.
   - `gh` covers source code, patch sets, and issue threads.
   - `web` covers official documentation and reference lists.
   - `reddit` covers field experience that no documentation carries.
- The points under a gap say WHAT you want out of that channel, not where it sits.
- A gap only you or the user can answer names that person instead of a channel.

**Template**

```
Gap 1 — <gap in one line> — gh
- <what you want out of gh>
- <what you want out of gh>

Gap 2 — <gap in one line> — web
- <what you want out of the web>

Gap 3 — <gap in one line> — reddit
- <what you want out of reddit>
```

🛑 STOP — Ask for remarks.

### Step 4 — Deliverables & Milestones

**Steps 2 and 3 culminate here in WHAT gets done and HOW.**
- First state the whole as one coherent picture, which may stay abstract.
- Then decompose it into ordered milestones.
   - A milestone is a logically delimited unit, independently committable and verifiable, ending in a deliverable.
   - Each deliverable states what is done and how to verify it.
   - Verification looks like a test command or an output match.
   - Code review does not count as verification.
- For a visual or live feature the user is the verifier, as the last gate.
   - That gate applies when self-verification by you is impossible.
- Present in chat the overall picture and the milestone sequence.
   - Per deliverable, present its verification and the affected file categories.

🛑 STOP — Ask for remarks.

---

## Phase 2 — Implement (after at least one worker is spawned)

### Step 1 — Dispatch

**Dispatch ONE milestone at a time, never the whole plan.**
- The milestones come from the Step 4 decomposition.
   - A small single-file fix is just one un-split milestone.
- Hand the worker the milestone as an abstract task plus the named files.
- The worker plans and reports back on its own, because that is its standing behavior.
   - Evaluate the returned plan at Step 2 before giving Go.
- Sign off on each milestone before dispatching the next.

**Stage 1, the integration branch.**
- Workers merge onto `integration` and never onto `main`.

1. The session starts on `main`, so run `git checkout -b integration` or switch to the existing one.
2. When switching to an existing integration branch, the branch-state check is mandatory. Run `git -C <repo> log integration..main --oneline | head -10`. A non-empty result means integration is behind main, and workers would spawn on stale code. Resolve it before spawning, by rebasing integration onto main or by merging main into integration. Staying on stale integration needs explicit user OK.
3. Workers spawn, and their worktrees branch from `integration`.
4. `worker-cli merge` merges into `integration`.
5. At session end, `git checkout main && git merge integration` syncs integration into main.

**Stage 2, prompt structure and spawn.**
- The prompt describes WHAT, and the worker figures out HOW.
- Every prompt matches exactly what was agreed with the user.
   - Extras along the way and variables the user did not ask for are not allowed.

| MUST include | MUST NOT include |
|---|---|
| The task described abstractly, meaning the problem and the desired outcome. | Exact code to write. The worker figures out its own implementation. External reference code from outside the project is the one exception, and you provide it. |
| The files and directories you found definitely relevant. They are a starting set and not a fence. Add any process-docs entries the worker should read for context. | Root cause hypotheses stated as facts. |
| The worktree path as workspace, phrased like "Your worktree is `<project>/.claude/worktrees/<name>/`. Read, edit, test, and commit here." | Implementation details that constrain the worker's approach. |
| The explicit negative scope, phrased like "Do NOT add features or improvements beyond the listed deliverables." | |
| The task-specific Completion Checklist items, meaning the verification points the worker outputs when done. | |
| The sentence "You are a WORKER." | |

Then spawn:
1. Write the prompt to `/tmp/spawn-worker-<project>-<name>.md`.
2. Run `worker-cli spawn <name> <prompt_file> <project_path> [model]`. The worktree is the default, so omit `--no-worktree`.
3. Immediately arm the wake-up, in the form the Wake-up Loop describes.

### Step 2 — Evaluate

**Compare the worker's plan against your own mental model from Phase 1.**
- After dispatching, the worker reads files in the worktree and reports findings plus approach.
   - Read the report via `worker-cli response`.
- Check for the same root cause, the same target files, and the same approach.
- On convergence, send "Go, implement it."
- On divergence of any kind, it is your turn to check.
   - Judge whether the worker's deviation from your mental model is actually right.
   - If it is right, give Go.
   - If it is wrong, send exactly where and why, and stay at Step 2.
- Accepting worker proposals at face value is prohibited.
   - Waving a plan through with "looks good" is prohibited too.

### Step 3 — Go + Implementation

- The worker implements after receiving Go.

### Step 4 — Review

**After the worker goes idle, review BEFORE merging.**

#### Code Review (MANDATORY)

1. Run `worker-cli response <name>`.
2. Read the worker's complete diff via Bash, and never via the Read tool on worktree paths. The canonical command is:
   ```bash
   git -C <project_root>/.claude/worktrees/<name> diff integration
   ```
   Do not restrict the diff to the last commit, because code review means reading the entire delta. For a single file's current content, use `git -C <worktree> show HEAD:<relpath>` or `cat` via Bash.
3. Check correctness, adherence to existing patterns, and absence of regressions.
4. If issues are found, treat them as a review disagreement.
5. If the review passes, proceed to Step 5.

**The review is non-skippable, even for ad-hoc or one-line merges.**
- Before every `worker-cli merge`, ask yourself whether you ran and read the diff in this session.
   - If not, stop and run the diff first.

**Sample-test rendered output for user-visible features, mandatory.**
- The rule applies when the feature affects formatted output, like search results or reports.
- Run one live sample and inspect the rendered text the user sees.
   - Reading the parser code that produces it does not count as a sample test.
- You run what you can.
   - For a live or visual verify, the user is the final gate.

**Interpretation cross-check when worker output interprets measured data, mandatory.**
- Workers often go beyond raw measurements and propose a mechanism for an observation.
- Before accepting such an interpretation, walk the four steps below.

1. Identify each interpretation claim, meaning a sentence like "this measurement proves X" or "the mechanism is Y".
2. Read the source code that produced the observation, so you know how the number was generated.
3. Ask whether a different mechanism could produce the same observation. If yes, the interpretation is one hypothesis among several. Either accept it as one candidate, or send the worker a follow-up probe that discriminates between the hypotheses.
4. Reject the interpretation but accept the data. The data stays valid evidence, while the unproven conclusion does not become the basis for the next worker task.

#### Review Disagreements

**A review disagreement is handled exactly like a Step 2 divergence.**
- The same check applies, and you prescribe no patch.

### Step 5 — Recap (MANDATORY after every milestone)

**After Step 4 completes clean, YOU send the recap trigger.**
- Send `worker-cli send <name> "recap"` after every milestone, without exception.
- The trigger is yours, and the worker runs its own recap pass scoped to its milestone.
- The recap consolidates the DOCS.md update and the process-docs entry into one commit.
   - It happens now, because the worker still has the task context in its head.
- If the worker dies mid-recap, you finish the recap yourself.
- Deferring documentation drift to the session-end recap is not allowed.

**The output is one recap commit, folded into the merge.**
- The worker commits one recap commit named `docs: recap for <task>`.
   - It reports the touched files and the doc updates.

### Step 6 — Merge

**Copy out what lives only in the worktree, before merging.**
- Gitignored files and extracted configs exist only in the worktree.
   - The merge deletes the worktree, so such files would be lost.

**`worker-cli merge <name> [project_path]` merges the branch into the current branch.**
- The current branch is `integration`, and the worker stays alive.
- For a cross-project worker, `project_path` is mandatory.

**Post-merge verification, mandatory.**
- If the merge says "Already up to date", stop.
   - For a cross-project worker, re-run the merge with `project_path`.
   - Otherwise the worker did not commit, so investigate via `worker-cli capture`.
- Run `git diff ORIG_HEAD --name-only` and check that the expected files are modified.
   - `ORIG_HEAD` is the pre-merge tip, so this shows all of the worker's commits.
   - A diff against `HEAD~1` would miss earlier commits on a multi-commit branch.
- If no changes show, send the worker commit instructions.

---

## Session Recap

**The session recap runs at the very end, only on the user's explicit trigger.**
- It is decoupled from the worker cycle, and the user decides when it happens.
- Never ask for it or propose it.
   - An ongoing session is the default state and needs no resolution.

**Your session recap covers ONLY files you touched directly.**
- Everything a worker produced is already in its own milestone recap.

### Phase 1 — RECAP 🔍

**The issues evaluation covers only issues touched this session.**
- Leave the rest untouched.
- For each touched issue, decide between closing and keeping it open.
   - Closing requires the work done and verified.
- Create a new issue only for a standalone task that surfaced this session and stays open.
- Update an issue body only if the issue's area changed, which is rare.

**Empty plate, capture every un-executed open item before closing.**
- Every open item from the original plan that was not executed gets captured.
   - Usually that capture is a process-docs entry.
   - An issue is right only when the item is a standalone task in its own right.

**Present the chat summary.**
- Name the issues touched, created, and closed this session.
- Name the doc files that get written or edited in the improve phase.

🛑 STOP — Ask for remarks.

### Phase 2 — IMPROVE+CLOSE 🛠️

**One run through, without stops.**
1. Execute the chat summary, so write the named doc files and do the issue hygiene exactly as presented.
2. Sync docs to RAG with `[ -f .rag-docs.json ] && rag-cli update_docs .`. The command skips silently when no manifest exists. RAG sync runs only here and never mid-session.
3. Close git for every repo this session touched, including cross-project targets. Per repo, run `git checkout main && git merge integration`, then `gcommit "<message>"`, then push. Before pushing, check for `.claude-plugin/plugin.json`. If it exists, use `plugin-publish`, and otherwise use `git push`.

- Done when the commits are pushed.
