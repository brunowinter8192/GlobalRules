# Workers (continued)

## Phase 2: Evaluate — Cross-Model Comparison

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

Concrete failure (2026-04-05): hooks-redesign worker implemented noise-filter and persisted-file-loading — both valid features, but neither addressed the core problem. Opus had no mental model and couldn't recognize the misalignment.

### Course Correction

If the worker's findings or delivered work is misaligned with the task:

1. `worker_capture` → read what they did or concluded
2. Identify the gap between expected and actual
3. `worker_send` with specific correction: "You did X but the requirement is Y. Please change Z."
4. NOT: spawn a new worker (wastes context, loses the worker's understanding)

Concrete failure (2026-04-04): Spawned 6 workers for hooks-pane UI. Should have been 1-2 workers with `worker_send` for corrections. Each new worker lost the previous worker's context.

**Multi-Source Bug Redirects — Name All Code Paths.** When redirecting a worker to fix a bug whose symptom can originate from MULTIPLE code paths (engine parser + downstream fetcher + renderer fallback), the redirect MUST enumerate ALL paths that could produce the symptom — not just the most-likely root cause. A worker given "fix in `openalex.py._reconstruct_abstract`" fixes exactly that file. If the bug ALSO surfaces from `preview.py._fetch_one` for the same data class, the worker won't see it without explicit naming. Test before sending: "Could this symptom appear via any code path I haven't named?" If yes — name it.

Concrete failure (2026-05-04, searxng bead-a45-impl): redirect named only `openalex.py._reconstruct_abstract`. Bug had two root causes — (a) openalex inverted-index join without `html.unescape`, (b) `preview.py._fetch_one` og:description extraction from Springer URLs also entity-encoded. Worker put fix in wrong file. Result: unfixed bug + follow-up bead + re-spawn cost next session.

**Per-Iteration User Status — debug loops need visibility.** When a worker goes through ≥3 internal iterations within a single task (debug loops, corrections, re-attempts), Opus posts a 2-line status update to the user at each iteration boundary — not only at task-completion. Format: **Hypothesis** (one sentence, what we think is wrong / what we're trying), **Result** (one sentence, what the test showed), **Next** (one sentence, what we do based on result, or "STOP for review"). Required for any task where the worker spawns diagnostic scripts, re-runs with changed config, or backtracks on a hypothesis. Each backtrack = one status update. Not required for single-iteration tasks (Phase 2 → Phase 3 → done is enough with a single "dispatched, waiting" + "done, here's the result").

Concrete failure (2026-04-21, searxng Google 30/30): 4 worker iterations (selector port → cookie inject → cookie scope fix → parse_js IIFE fix). User saw only timer pings between "0/30 baseline" and "30/30 OK". User asked mid-session "ok was hat der worker denn jetzt konkret gemacht?" — signal that the blackbox was too opaque. Per-iteration 2-line status would have kept the narrative visible.

---

## Phase 3: Go + Implementation

Worker implements after receiving Go. During implementation:

### While Workers Run

**Do NOT poll status repeatedly.** Workers take 2-10 minutes.

**Pattern:**
1. Spawn worker → do independent work (rule edits, other planning, exploration)
2. Background timer fires → `worker_status` → if idle, proceed with Phase 4
3. Use `worker_capture(tail=N)` to read output after worker goes idle

### Timer & Polling Flow (NON-NEGOTIABLE)

Canonical flow, in this order, nothing else in between:

1. **Spawn worker.**
2. **Set ONE background timer** — `Bash(command="sleep N && echo done", run_in_background=true)`. `N` = estimated work duration, default ~180s (3min). For Phase-A-only investigation tasks: 60–120s. For full implement + test: 180–300s.
3. **Wait for the timer to fire.** Do independent work (rule edits, bead updates, reading unrelated code) — no tool calls related to this worker during the wait.
4. **Timer fires** (done-file non-empty, or hook notification) → `worker-cli status <name> <project_path>`.
5. **If `working`:** set the NEXT background timer, wait again. NO capture, NO cat on the output file, NO re-status-check with the same tool in the same response turn.
6. **If `idle`:** `worker-cli response <name> <project_path>` — returns clean text from the session JSONL without the CC UI trailers. Only fall back to `worker-cli capture` + tail + sed-filter if `response` returns something unexpected.

**Background Task Discipline (CORE INVARIANT):** maximum ONE background task — timer OR any other `run_in_background=true` Bash — in flight at any moment. The rule is not about context budget, it is about CC's event-queue: every background-task completion arriving while an API stream is open cancels that stream client-side and fires a new REQ with the updated payload. Two completions back-to-back during a stream = 2-fold abort cascade, billed for input + cache-read per aborted REQ. Idle state is safe — completions arriving when no REQ is streaming queue normally for the next turn.

**Foreground vs Background:** `sleep` and `until [ -s <file> ]; do sleep N; done` MUST run in `run_in_background=true`. Never chain a foreground sleep/until-loop next to an already-running background timer for the same wait — that is two "task-completions" back-to-back on the same completion-file, and it is redundant work. The blocking message "sleep X followed by: <command>" from tool-use is a signal to switch to background, not to "work around" with a shorter chain.

**No manual cat on timer output files.** `/private/tmp/claude-501/.../tasks/*.output` is checked by either the until-loop wait-condition or the worker-done hook. Reading it manually between polls returns zero new information and wastes a Bash call.

Cross-ref: `Monitor_CC/decisions/OldThemes/background_task_abort_cascade.md` has the full forensic + billing analysis of the mechanism, with case-1 evidence from session 1776977437.

**Post-Spawn-Ack — No Thinking, No Speculation:**

After spawning a `Bash(run_in_background=true)` timer, the next response is an acknowledgment-only single-line ("Timer läuft, ich warte." or equivalent). No reasoning about expected worker outputs, no orchestration planning in that turn.

Positive framing: when a worker returns results, think deeply about it. When starting a wait, do not think — task is dispatched, results require fresh thinking only when they return.

Broader principle: while ANY worker is `working`, no speculation about expected outputs in any context (visible response or thinking). Speculation = pre-thinking what a worker will produce, which (a) biases later evaluation of the actual report (cross-model verification requires independence), (b) consumes thinking budget that's invalidated when real output arrives, (c) extends post-spawn streams = abort-cascade risk window.

**Structural enforcement on top of the disciplinary rule:** A proxy-side rule auto-overrides `thinking.type=disabled` on REQs immediately following `Bash(run_in_background=true)` with `sleep` command. Bead `Monitor_CC-xo9` tracks implementation. Once active, the prompt rule above is the default discipline; the proxy override is the deterministic backup-cap that catches lapses.

Concrete failure (2026-04-26): bilp4nirg timer (3rd of 3 in session) caused a 5-minute aborted REQ#16 cascade because the post-spawn response was an extended-thinking turn (OUT=227, signature length high) reasoning about orchestration trajectory — while previous spawns had OUT=16 ack-only responses. ~$0.04 cost for the aborted REQ + 5-minute dead latency. See `decisions/OldThemes/background_task_abort_cascade.md` for full forensic.

Concrete failure (2026-04-22): Started sleep 45, sleep 30, sleep 30 within seconds of each other without waiting for any to complete. Each timer wake-up triggered another status check + another timer, creating a polling loop that wasted context and confused the user.

Concrete failure (2026-04-24, this session): During a worker run, Opus dispatched a `sleep 120` background timer, and while it was still running started a foreground `until [ -s <tmp>/<id>.output ]; do sleep 5; done` waiting on the SAME output file. Also called `worker-cli status` twice in 2 minutes without an intervening timer, plus a manual `cat` on the timer output file with no associated wait. User pointed out the full sequence as "zig fehler, alles scheiße executed". Root cause: no internalized polling-flow template — each idle moment triggered another ad-hoc check instead of "wait for timer, then single check, then next timer or read."

**Worker idle ≠ task complete.** `worker-cli status idle` only reflects tmux pane activity (10s window_activity threshold). When the worker spawned its OWN background sleep (e.g. `sleep 480 && echo done` to wait for an internal smoke run), tmux activity goes quiet → status reports idle → Opus thinks the user-visible task is done. Reality: worker is mid-task in its own sleep loop. Always read the LAST CONCRETE MESSAGE from `worker-cli response` before declaring a phase complete. If it references "waiting for sleep" / "on track for N min" / "background smoke running" — worker is NOT done. Either wait longer than the worker's internal timer, or `worker-cli send` with a wake message to force the next step.

Concrete failure (2026-05-03, searxng preview worker): worker dispatched `./venv/bin/python 05_search_smoke.py > /tmp/05_smoke.log &` then `sleep 480 && echo done`. tmux activity dropped during the 8-min sleep. `worker-cli status` returned `idle 56%`. `worker-cli response` showed only "5/30 in ~60s — on track for ~6min total. Waiting for the 480s timer." — worker NOT done. Required a wake-up send to force commit.

### Capture vs Status — Don't Capture While Working

Worker capture is EXPENSIVE — each call dumps the last N lines including the worker prompt echo (often 2k+ chars), CC UI trailers (`✻ Baked for Xm Ys`, `Composing…`, divider, `Sonnet | XX%`, `❯`, `⏵⏵ accept edits on`), and duplicate frames Sonnet re-renders.

**Rule — minimize captures:**

1. **Status check is cheap.** While a worker is `working`, only call `worker_status` (or `worker-cli status`). Do NOT capture. Set a timer, re-check status.
2. **Capture only when idle.** Fresh output is relevant when the worker has finished a chunk of work. Mid-thought snapshots are noise.
3. **Default capture size: small.** `tmux capture-pane -p -S -60 | tail -40` is the default for catching the last response. Raise only if you need more history.
4. **Pre-filter UI trailers.** Pipe capture through `sed -E '/^[│▁─]+$/d; /Sonnet \| [0-9]+%/d; /^[[:space:]]*❯[[:space:]]*$/d; /^[[:space:]]*⏵⏵ accept edits/d; /^✻.*for [0-9]+m/d; /^✢ Composing/d; /^· Symbioting/d; /Tip: /d'` to strip the CC UI noise. Expect 6-10 lines filtered per capture.
5. **Context-% visibility.** The only reason to capture while working is to see `Sonnet | XX%`. Prefer extracting this from `worker_status` output when available — don't scrape the whole pane just for the percentage.

Concrete failure (2026-04-19): Multiple captures with `-S -400 | tail -350` during Phase A of the tool-use-analysis and waste-pane workers. Result: ~5k+ chars of worker-prompt-echo, Phase-A-plans, and old tool outputs dumped into Opus context per check. User called it out: "definitiv nicht gut".

**Response Format Discriminator — intermediate vs final.** When `worker-cli response <name>` returns text that looks like a Phase-A plan or in-progress narrative ("Implementing.", "Reihenfolge:", "Files gelesen", "Now I will...", section headers without checklist), do NOT trust it as a "done" signal — even if the worker briefly went idle. Phase-A intermediate output and Completion Checklist output are syntactically distinct: the latter ends with a commit SHA, a `[x]` checklist, or a "Pre-Commit Live Checks" header; the former is open-ended prose. Test: does the response end with commit SHA / `[x]` / "Pre-Commit"? If no — re-check status with `worker-cli status` before merging.

Concrete failure (2026-04-28, Monitor_CC cap-fix worker, rkk Phase B): captured response showed "Jetzt alle relevanten Files gelesen. Vollständiges Bild: ... Implementing. Reihenfolge: addon.py → logging.py → ...". Reported as "done" to user. Status re-check showed `working 56%`. Worker actually finished 10 minutes later with the proper Completion Checklist + commit SHA `154c69e`.

**Quota-Limit Detection in Worker Capture.** When a worker capture surfaces strings like `"You're out of extra usage"`, `"out of extra usage"`, `"resets <time>"` — the worker's Anthropic billing quota is exhausted. The worker can either make no further LLM calls or only severely-rate-limited ones at high latency. Continuing to let the worker investigate burns quota for no progress and produces half-done outputs. Action when this string appears in any capture: `worker_send` IMMEDIATELY with "Stop current investigation. Commit current state with whatever message captures the work. Output completion checklist for what's done. Do NOT debug further." Treat any in-progress bug or partial-feature as a follow-up bead — do NOT keep the worker running on degraded quota chasing fixes. After commit lands, merge the partial work and proceed.

Concrete failure (2026-05-04, searxng bead-a45-impl worker): worker hit "You're out of extra usage · resets 9:10pm (Europe/Berlin)" mid-debug of an OpenAlex HTML-entity bug. Continued debugging for ~5 more minutes on quota-restricted mode with no commit, dropping to 22% context. Required explicit Opus redirect to "stop and commit". Should have been caught from the FIRST capture that surfaced the quota line.

### Permission Dialogs for Privileged Paths

Workers spawn with `accept edits on` but some paths still trigger confirm dialogs that block the worker mid-task:

- `~/.claude/**` (shared-rules, settings, plans, rules files)
- `~/.claude/plugins/cache/**` (plugin-sync target directories — especially `mkdir` for a new version, e.g. `mkdir ~/.claude/plugins/cache/brunowinter-plugins/<plugin>/<new_version>/`)
- `~/.gitconfig`, `~/.zshrc`, any dotfile outside the project tree
- Project `.claude/settings.local.json` edits

**Symptoms:** Worker goes `working` but produces no new output. `worker_capture` shows a prompt dialog with options `1. Yes / 2. Yes, and allow... / 3. No`.

**Mitigation:**
1. **In worker prompt (preemptive):** If the task edits `~/.claude/**` paths OR runs `plugin-sync.sh` with a new version bump (which mkdir's a new cache version dir), include a disclaimer: "You will hit a permission dialog on the first write to `~/.claude/...` (e.g. `mkdir ~/.claude/plugins/cache/brunowinter-plugins/<plugin>/<version>/`). This is expected — send `2` at the Claude Code prompt to grant session-wide permission."
2. **From Opus (fallback):** When `worker_status` shows `working` but capture is stuck on a dialog, use `tmux send-keys -t worker-<project>-<name> "2" Enter` to accept. `1` works too but re-prompts for every subsequent write.
3. **Verify unblocked:** after sending the key, re-check `worker_status` — should remain `working` as the worker proceeds to the next edit.

Concrete failure (2026-04-14): `inject-fixes` worker building the 6 CLI skills hit confirm dialogs for every write under `~/.claude/shared-rules/opus/` (cli-skills.md + 3 existing rule appends). Opus had to manually `tmux send-keys "1"` / `"2"` twice to unstick the worker. One preemptive disclaimer in the prompt would have eliminated both interventions.

Concrete failure (2026-04-14 evening): 3 of 4 plugin-conversion workers (`reddit-cli-convert`, `arxiv-cli-convert`, `rag-cli-convert`) blocked mid-task on the SAME dialog pattern: `mkdir ~/.claude/plugins/cache/brunowinter-plugins/<plugin>/1.1.0/` (new plugin version bump during `plugin-sync.sh`). Opus had to `tmux send-keys "1"`/`"2"` on each. One preemptive disclaimer in the shared conversion-prompt template would have eliminated all 3 interventions.

---

## Phase 4: Review

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

**Why no Read tool on worktree paths:** every worktree contains its own `CLAUDE.md` (CC treats worktrees as first-class project checkouts). When Opus invokes the Read tool on any file under `.claude/worktrees/<name>/...`, CC re-injects the worktree's CLAUDE.md as a system-reminder into the current turn — duplicating the project-CLAUDE.md that was already loaded at session start. Bash `cat` / `git show` / `git diff` return file bytes directly without triggering the CLAUDE.md injection, so they are strictly cheaper for worktree file access. The Read tool remains the default for files in the main project tree; only worktree paths (`/.claude/worktrees/...`) need the bash-only rule.

Concrete failure (2026-04-24): During Phase 4 review of the req-cascade-doc worker, Opus used `Read /Users/.../Monitor_CC/.claude/worktrees/req-cascade-doc/decisions/OldThemes/background_task_abort_cascade.md`. The worktree's CLAUDE.md fired as a fresh system-reminder on that turn. User flagged this post-session as a repeatable cost — rule lifted to always-diff/always-bash for worktree reads.

**Sample-Test rendered output (MANDATORY for user-visible features).** When the feature affects formatted output (search results, reports, CLI display, generated text): run ONE live sample (single query, single test case) and inspect the rendered text — not the JS or parser code that produces it, the actual string the user sees. Look for redundant fields, concatenation artifacts, broken truncation, inconsistent spacing, button text bleeding into content. Code-read of selector or parser logic does NOT count as sample-test — the selector might be syntactically valid yet match a wider DOM region than intended. Bug class "stale extraction" is only catchable here.

Concrete failure (2026-05-03, searxng DDG + preview merge): worker shipped DDG engine + preview feature + 05_search_smoke. Phase 4 review read the diff, confirmed structure mirrors google.py, merged. User reviewed report later, immediately noticed Google snippets contained title + URL breadcrumb + "Web results" header + "Read more" button → bloated 6-source-concatenation. Visible only by reading the rendered output, not the JS source. Pre-existed in google.py JS_PARSE selectors but unnoticed because no Phase 4 ever sample-tested the rendered snippet text.

### Worker-Statement vor Fix (Standard)

When code review finds an issue: **ask the worker what they think** before prescribing a fix. Pattern:
1. `worker_send`: "Review-Frage: [describe issue]. Was ist dein Statement dazu?"
2. Worker analyzes and responds with their assessment
3. Opus evaluates the worker's statement — worker may confirm, deny, or reveal deeper issues
4. THEN send fix instructions based on the combined understanding

**Why:** The worker has implementation context that Opus lacks. Asking for a statement often reveals issues Opus didn't see (e.g., wrong field names, missing edge cases). Blind fix instructions based on Opus's assumptions lead to wrong fixes.

Concrete failure (2026-04-10): Opus prescribed api_call field names `usage.cache_read_input_tokens` — wrong. Worker's statement revealed the actual fields are `cache_read`, `cache_creation`, `direct`. One question saved a broken commit.

**What to check:**
- Does the code address the actual problem (not a symptom)?
- Does it follow existing patterns in the codebase?
- Are there uncommitted local changes that conflict?
- Did the worker commit? (Check Completion Checklist)

**Glue work before merge:** Copy gitignored files, extract configs — anything that lives only in the worktree. Merge deletes the worktree, anything not saved is lost.

Concrete failure (2026-03-25): engagement-agent Worker rewrote `.claude/agents/engagement-finder.md` (gitignored). Worktree deleted on merge. File had to be rescued via /tmp copy.
