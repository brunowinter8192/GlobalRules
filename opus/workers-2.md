# Workers (continued)

## Worker Phase 2: Evaluate — Cross-Model Comparison

After dispatching, the worker reads files in the worktree and reports findings + proposed approach. Read via `worker_capture`.

**Compare the worker's findings against your own mental model from PLAN:**
- Do worker and Opus identify the SAME root cause / same target files / same approach?
- If the worker challenges Opus with a BETTER idea → evaluate honestly, update your model.
- If the worker misses something Opus sees → correct (see Course Correction below).
- If Opus and worker diverge on root cause → iterate investigation; do NOT give Go yet.

**Decision gate:**
- Convergence → `worker_send`: "Go, implement it."
- Divergence on scope/approach → `worker_send` with correction.
- Divergence on root cause → `worker_send` asking for deeper investigation with specific questions. Stay at Phase 2.

**Prohibited:**
- Accepting worker proposals at face value without comparing to your mental model
- "Looks good" rubber-stamping
- Skipping Phase 2 and letting the worker proceed straight to implementation
- Merging at Phase 5 after reading only the Completion Checklist without reviewing the actual code

### Follow-up dispatches use the same gate

The investigate-report-STOP-Go cycle applies to EVERY `worker_send` carrying a fix/change, not just initial spawn. Ad-hoc fixes are exactly where cross-model verification has the highest value — Opus formed the hypothesis under pressure and is most likely wrong.

In follow-up prompts: describe the SYMPTOM and provide diagnostic data, not the solution. Include the STOP gate (same sentinel as initial spawn). Wait for the worker's report → cross-model compare → only then Go.

Forbidden phrasings: "Apply this fix: <patch>" / "The bug is in line N, change A to B" / "I traced it to <root cause>. Make this change." Anything that hands over the conclusion turns the worker into Opus's typist and defeats the verification.

### Course Correction

If the worker's findings or delivered work is misaligned with the task:

1. `worker_capture` → read what they did or concluded
2. Identify the gap between expected and actual
3. `worker_send` with specific correction: "You did X but the requirement is Y. Please change Z."
4. NOT: spawn a new worker (wastes context, loses the worker's understanding)


**Multi-Source Bug Redirects — Name All Code Paths.** When redirecting a worker to fix a bug whose symptom can originate from MULTIPLE code paths (engine parser + downstream fetcher + renderer fallback), the redirect MUST enumerate ALL paths that could produce the symptom — not just the most-likely root cause. A worker given "fix in `openalex.py._reconstruct_abstract`" fixes exactly that file. If the bug ALSO surfaces from `preview.py._fetch_one` for the same data class, the worker won't see it without explicit naming. Test before sending: "Could this symptom appear via any code path I haven't named?" If yes — name it.


**Per-Iteration User Status — debug loops need visibility.** When a worker goes through ≥3 internal iterations within a single task (debug loops, corrections, re-attempts), Opus posts a 2-line status update to the user at each iteration boundary — not only at task-completion. Format: **Hypothesis** (one sentence, what we think is wrong / what we're trying), **Result** (one sentence, what the test showed), **Next** (one sentence, what we do based on result, or "STOP for review"). Required for any task where the worker spawns diagnostic scripts, re-runs with changed config, or backtracks on a hypothesis. Each backtrack = one status update. Not required for single-iteration tasks (Phase 2 → Phase 3 → done is enough with a single "dispatched, waiting" + "done, here's the result").


---

## Worker Phase 3: Go + Implementation

Worker implements after receiving Go. During implementation:

### While Workers Run

**Do NOT poll status repeatedly.** Workers take 2-10 minutes.

**Pattern:**
1. Spawn worker → do independent work (rule edits, other planning, exploration)
2. Background timer fires → `worker_status` → if idle, proceed with Phase 4
3. Use `worker_capture(tail=N)` to read output after worker goes idle

### Timer & Polling Flow (NON-NEGOTIABLE)

Timers exist for ONE reason: waking Opus periodically to check progress on a `working` worker. They have NO other legitimate use.

1. **Spawn worker.**
2. **If worker becomes `working`** (active tool calls): set 10min timer `Bash(command="sleep 600 && echo done", run_in_background=true)`.
3. **Timer wakes → `worker-cli status <name>`.** `working` → new 10min timer. `idle` → `worker-cli response <name>` (fallback: `worker-cli capture` + tail + sed-filter), proceed to next phase.

**Worker `idle` ≠ timer needed.** When the worker is `idle` and Opus has no further immediate action queued — e.g., waiting on user review of intermediate output, waiting on a long-running external process that the worker spawned and then went idle (sweep, indexing, smoke run, background compute) — Opus does NOT set a timer. Opus goes truly idle itself and waits for the user to wake it.

Brief chat hint to user before going idle: "`<task>` läuft (geschätzt ~X min), Worker steht idle bei Y% Context. Weck mich wenn du denkst es ist fertig" or equivalent. The user pings manually via "background done" / "check now" / explicit re-prompt. ONLY THEN Opus checks status (filesystem for the external task's output, `worker-cli status` for the worker).

Rationale: a timer that wakes Opus only to find "still waiting" is pure waste — it consumes Opus context with no progressable action, since the next legitimate step requires either the external process to finish (a filesystem signal, not timer-pollable) OR user input (also not timer-pollable). Going truly idle preserves Opus context for the actual next orchestration step.

The single exception remains worker-in-`working`-state: there a periodic status check via timer is the only mechanism to detect when the worker becomes `idle`.

**Background Task Discipline:** maximum ONE background task — timer OR any other `run_in_background=true` Bash — in flight at any moment.

**Foreground vs Background:** `sleep` and `until [ -s <file> ]; do sleep N; done` MUST run in `run_in_background=true`. Never chain a foreground sleep/until-loop next to an already-running background timer.

**No manual cat on timer output files.**

**Post-Spawn-Ack — No Thinking, No Speculation.** After spawning a `Bash(run_in_background=true)` timer, the next response is acknowledgment-only single-line ("Timer läuft, ich warte." or equivalent). No reasoning about expected worker outputs, no orchestration planning in that turn. While ANY worker is `working`, no speculation about expected outputs in any context.

A proxy-side rule auto-overrides `thinking.type=disabled` on REQs immediately following `Bash(run_in_background=true)` with `sleep` as a structural backup.


**Worker idle ≠ task complete.** `worker-cli status idle` reflects tmux pane activity only (10s threshold). When the worker spawned its own background sleep, tmux goes quiet but worker is mid-task. Always read the LAST CONCRETE MESSAGE from `worker-cli response` before declaring a phase complete. If it references "waiting for sleep" / "background smoke running" — worker is NOT done. Wait longer or `worker-cli send` a wake message.


### Capture vs Status — Don't Capture While Working

Worker capture dumps prompt echo + CC UI trailers + duplicate frames (often 2k+ chars). Minimize captures:

1. **Status check is cheap.** While a worker is `working`, only call `worker_status`. Do NOT capture. Set a timer, re-check status.
2. **Capture only when idle.**
3. **Default capture size small:** `tmux capture-pane -p -S -60 | tail -40`. Raise only if you need more history.
4. **Pre-filter UI trailers** via `sed -E '/^[│▁─]+$/d; /Sonnet \| [0-9]+%/d; /^[[:space:]]*❯[[:space:]]*$/d; /^[[:space:]]*⏵⏵ accept edits/d; /^✻.*for [0-9]+m/d; /^✢ Composing/d; /^· Symbioting/d; /Tip: /d'`.
5. **Context-% visibility.** Only reason to capture while working — prefer `worker_status` output if available.


**Response Format Discriminator — intermediate vs final.** When `worker-cli response <name>` returns text that looks like a Phase-A plan or in-progress narrative ("Implementing.", "Reihenfolge:", "Files gelesen", "Now I will...", section headers without checklist), do NOT trust it as a "done" signal — even if the worker briefly went idle. Phase-A intermediate output and Completion Checklist output are syntactically distinct: the latter ends with a commit SHA, a `[x]` checklist, or a "Pre-Commit Live Checks" header; the former is open-ended prose. Test: does the response end with commit SHA / `[x]` / "Pre-Commit"? If no — re-check status with `worker-cli status` before merging.


**Quota-Limit Detection in Worker Capture.** When a worker capture surfaces strings like `"You're out of extra usage"`, `"out of extra usage"`, `"resets <time>"` — the worker's Anthropic billing quota is exhausted. The worker can either make no further LLM calls or only severely-rate-limited ones at high latency. Continuing to let the worker investigate burns quota for no progress and produces half-done outputs. Action when this string appears in any capture: `worker_send` IMMEDIATELY with "Stop current investigation. Commit current state with whatever message captures the work. Output completion checklist for what's done. Do NOT debug further." Treat any in-progress bug or partial-feature as a follow-up bead — do NOT keep the worker running on degraded quota chasing fixes. After commit lands, merge the partial work and proceed.


### Permission Dialogs for Privileged Paths

Workers spawn with `accept edits on` but some paths still trigger confirm dialogs that block the worker mid-task:

- `~/.claude/**` (shared-rules, settings, plans, rules files)
- `~/.claude/plugins/cache/**` (plugin-publish target directories — especially `mkdir` for a new version)
- `~/.gitconfig`, `~/.zshrc`, any dotfile outside the project tree
- Project `.claude/settings.local.json` edits

**Symptoms:** Worker goes `working` but produces no new output. `worker_capture` shows a prompt dialog with options `1. Yes / 2. Yes, and allow... / 3. No`.

**Mitigation:**
1. **In worker prompt (preemptive):** If the task edits `~/.claude/**` paths OR runs `plugin-publish` with a version bump (which mkdir's a new cache version dir), include a disclaimer: "You will hit a permission dialog on the first write to `~/.claude/...`. This is expected — send `2` at the Claude Code prompt to grant session-wide permission."
2. **From Opus (fallback):** When `worker_status` shows `working` but capture is stuck on a dialog, use `tmux send-keys -t worker-<project>-<name> "2" Enter` to accept. `1` works too but re-prompts for every subsequent write.
3. **Verify unblocked:** after sending the key, re-check `worker_status` — should remain `working` as the worker proceeds to the next edit.


### File-Move Task Checklist

When a worker task involves moving files to a new subdirectory, the worker must verify EACH of the following AFTER completing the move. Add this checklist explicitly to the worker prompt:

> **File-Move Checklist** — verify EACH point after completing the move:
> 1. **Imports inside moved file:** `.` / `..` prefix depth changed (e.g. `from . import x` → `from .. import x` if now one level deeper). Fix all.
> 2. **Imports outside referencing the moved file:** every caller that imports the old path must be updated.
> 3. **Lazy imports inside functions:** `from . import x` written INSIDE a function body is still a relative import and follows the same rule. Easy to miss — grep explicitly.
> 4. **Grep verification:** `grep -rn 'from \.\|from \.\.' <affected_subdirs> | grep <moved_module_name>` — confirms all references are updated.
> 5. **Smoke test:** run the entry-point or a targeted import check (`python -c "import <top_level_package>"`) to confirm no ModuleNotFoundError.

The checklist is mirrored on the worker side in `~/.claude/shared-rules/worker/worker-rules.md` Section 3 (File-Move Checklist subsection) — so it fires via proxy-injected rules even if Opus forgets to echo it in the prompt. Add it to BOTH places when this rule is updated.


### Scope Extension During IMPLEMENT

When the user introduces a new scope during IMPLEMENT:

**Before each step below: RAG query on the affected module/topic (`rag-cli search_hybrid "<topic>" <Project>-meta`). Mini-scoping waives PLAN structure, NOT RAG-First (see workers-1 § RAG-First on Any Project Question).**

Mini-scoping (no full PLAN Phase needed):
1. Summarize in chat: what is the user's task, what would a worker do
2. Check `worker_list` — is there an alive worker with context overlap? Default to `worker_send` on that worker (AGGRESSIVE REUSE, workers-3). Only spawn fresh if no candidate fits.
3. Dispatch if user has no remarks — investigate-report-stop pattern still applies (see Worker Phase 1 Prompt Structure in workers-1).


---

## Worker Phase 4: Review

After worker goes idle, review BEFORE merging.

### Code Review (MANDATORY)

1. `worker-cli response <name>` (or `worker_capture` + tail as fallback) → read Completion Checklist
2. Read the worker's complete diff via Bash, NEVER via the Read tool on worktree paths. Canonical command:
   ```bash
   git -C <project_root>/.claude/worktrees/<name> diff dev
   ```
   This shows the full diff from dev tip to the worker's branch tip — every change the worker made, including across multiple commits on the branch. Do NOT restrict to `HEAD~1..HEAD` or `dev..HEAD` (the latter is equivalent but with redundant `HEAD`); code review means reading the entire delta, not only the last commit. For a single file's current content: `git -C <worktree> show HEAD:<relpath>` or `cat <worktree>/<relpath>` via Bash.
3. Check: correctness, existing patterns followed, no regressions
4. If issues found → ask worker for statement (see Worker-Statement vor Fix below)
5. If review passes → proceed to Phase 5

**Non-skippable — even for ad-hoc / one-line / context-recovery merges.** Self-test before EVERY `worker-cli merge`: "Have I run `git -C <worktree> diff dev --` and READ the result in this session?" If no → STOP, run the diff first.

**Sample-Test rendered output (MANDATORY for user-visible features).** When the feature affects formatted output (search results, reports, CLI display, generated text): run ONE live sample and inspect the rendered text — not the parser code that produces it, the actual string the user sees. Code-read does NOT count as sample-test.

**Interpretation Cross-Check (MANDATORY when worker output contains an interpretation of measured data).** Investigation workers often deliver findings narratives that go beyond raw measurements — they interpret the data and propose a mechanism ("data X means mechanism Y"). Before accepting the interpretation:

1. Identify each interpretation claim — a sentence of the form "this measurement means/proves/shows X" or "mechanism is Y".
2. For each claim, locate the source code that produced the data being interpreted. This must be the actual code, in the current src/ tree, read in this session (Step 2 Stage 3 should already have covered it — if not, read it now BEFORE accepting the interpretation).
3. Ask: are there alternative code paths in the same function/module that would produce the same measurement but support a DIFFERENT interpretation? If yes, the worker's interpretation is one of several possible — not proven. Either accept it as one hypothesis among several, or send the worker back with a follow-up probe that discriminates between the candidates.
4. **Reject the interpretation, accept the data.** If the worker's interpretation does not uniquely follow from the source code, the data they collected is still valid evidence — but the conclusion they drew is not yet supported. Phase 5/6 may still proceed (merge the probe artifacts), but the interpretation does NOT become the basis for the next worker's task.

The canonical failure mode this prevents: Worker collects clean data, draws Interpretation A (which fits the most salient prior hypothesis), Opus reviews diff at code-shape level only, accepts Interpretation A, scopes next worker on the basis of A — and one user challenge about the underlying code reveals that Interpretation B (also consistent with the data, also in the source) was equally possible and the entire chain was inference-stacked.

### Worker-Statement vor Fix

When code review finds an issue: ask the worker what they think before prescribing a fix.

1. `worker_send`: "Review-Frage: [describe issue]. Was ist dein Statement dazu?"
2. Worker analyzes and responds
3. Opus evaluates the statement — worker may confirm, deny, or reveal deeper issues
4. THEN send fix instructions based on the combined understanding


**What to check:**
- Does the code address the actual problem (not a symptom)?
- Does it follow existing patterns in the codebase?
- Are there uncommitted local changes that conflict?
- Did the worker commit? (Check Completion Checklist)

**Glue work before merge:** Copy gitignored files, extract configs — anything that lives only in the worktree. Merge deletes the worktree, anything not saved is lost.


---

## Worker Phase 5: Recap (MANDATORY After Every Stage)

**Trigger:** ALWAYS — after Phase 4 Review completes clean for ANY task / etappe, Opus sends `worker_send <name> "recap"`. Non-discretionary. The recap consolidates DOCS.md sync, decisions/ IST consistency, and OldThemes persistence into ONE commit while the worker still has the original task context in head.

**Why mandatory:** drift accumulation across multiple stages compounds. If Opus skips recap to "save context for next task", the drift sits in the worker's worktree until session-end RECAP — where Opus-side archaeology must reconstruct what changed from git log + file state. Worker-side recap with original task context is cheaper (worker remembers exact intent) AND more accurate (no inference required). Recap-after-every-stage also makes the worker's committed state ALWAYS up-to-date, so when a worker dies mid-next-task, the successor inherits clean state.

**Sequence per task-cycle:**
1. Worker delivers Completion Checklist (idle)
2. Opus runs Phase 4 Review (diff inspection, sample-test, interpretation cross-check)
3. Opus sends `worker_send <name> "recap"` — MANDATORY, no skip
4. Worker commits ONE `docs: recap for <task>` commit
5. Opus runs Phase 6 Merge
6. THEN Opus decides next dispatch per workers-3.md § AGGRESSIVE REUSE (thematic continuity, NOT context-budget)

**Only exception — session ending immediately:** if Opus is already in the IMPROVE+CLOSE phase and the worker just completed its final task, the session-end RECAP absorbs the cleanup. Mid-session, recap-after-every-stage is the rule.

**Recap on low-context workers — always send recap, worker handles handoff.** Even at <20% remaining, send `recap`. The recap pattern itself (including partial-recap + SUCCESSOR-HANDOFF when context is too tight) is fully defined in `~/.claude/shared-rules/worker/worker-rules.md` § 6 — Opus does not duplicate the format spec, does not embed it in worker prompts. Opus just sends the trigger `recap`.

If the worker dies mid-recap, the partial-recap commit on its branch carries the SUCCESSOR-HANDOFF note. Opus spawns a successor with: "Read latest commit on branch `<dying-worker-branch>` — it contains a SUCCESSOR-HANDOFF. Resume from there." Successor finishes the recap as its first task. Opus does NOT do file archaeology and NEVER defers drift to session-end RECAP.

**Phase 5 output:** worker commits ONE recap commit (`docs: recap for <task>`), reports drift counts pre/post + touched-file list. Folds into Phase 6 Merge.

