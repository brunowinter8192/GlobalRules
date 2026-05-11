# 2026-05-06 — SEARXNG: scrape skill prod migration

Session built `cleanup-and-index` skill (worker-facing) + extended `web-research` skill (user-facing), prod-migrated PDF auto-routing + 31z minimal_content + strip_bloat regex, refactored explore_site, closed 6 beads. Four process lessons surfaced.

---

## ~/.claude/shared-rules/opus/workers-3.md → "Pre-followup Branch Sync" subsection

### Add companion paragraph: Mid-Worker-Flight Opus Commits

When Opus makes NEW commits to dev BETWEEN worker dispatch and worker completion, the worker's working tree (synced at dispatch time via Pre-followup Branch Sync) does NOT see these commits. Worker's resulting diff vs dev shows misleading "added back" or "modified" entries for files that Opus changed mid-flight. Sometimes 3-way merge auto-resolves cleanly via common-ancestor analysis (when worker didn't touch the same files); sometimes it produces conflicts.

**Rule:** After dispatching a worker, if Opus commits ANY new changes to dev before the worker reports idle, immediately `worker-cli send <name> "Re-sync: I committed <SHA> mid-flight. git -C <worktree> fetch origin dev && git -C <worktree> merge dev before continuing."`. Don't wait for the worker to finish and discover the drift in their diff.

**The test:** between every Opus-side `git commit` on dev and the next `worker-cli merge`, ask: "is any worker still mid-task on this branch?" If yes → re-sync notify required.

Concrete failure (2026-05-06, searxng cleanup-and-index session): Opus made 2 mid-flight commits to dev (skill additions `987675c` + `d4c7b32`, then agent_pipeline deletion `4c564e8`) without notifying the active strip-bloat-fix worker. Worker's diff vs dev showed `dev/agent_pipeline/DOCS.md` as `A` (added) plus extra agent_pipeline lines in `dev/DOCS.md` and `CLAUDE.md` that the worker hadn't actually written. 3-way merge auto-resolved correctly (worker hadn't modified those files, so dev's deletion won), but it required mental load to read the diff carefully and verify merge would not conflict. A 5-second worker-cli send re-sync notification would have eliminated the entire confusion.

---

## ~/.claude/shared-rules/opus/skills.md (NEW FILE) → Skill Design

### Audience-First Design

Two fundamentally different skill categories. The structural and content rules differ.

**User-facing skill** (loaded by Opus during main session):
- Describes a USER WORKFLOW that the user triggers naturally or explicitly
- Phases involve user decisions (which URLs to keep, which PDFs to convert, which collection name)
- References CLI commands as instructions to Claude
- IF the workflow ends with a worker dispatch: includes the SHORT spawn-prompt pattern that Opus writes (~10 lines: identity + inputs + "activate skill X")
- The methodology stays OUT — methodology is in the worker-facing skill the worker activates

**Worker-facing skill** (loaded by a worker via `Skill(skill="<name>")` after spawn):
- Describes TECHNICAL METHODOLOGY the worker follows autonomously
- Phases are mechanical execution steps, no mid-flight user input
- References CLI commands as direct invocations
- NEVER contains a worker-spawn template (worker doesn't spawn other workers in this pattern)
- Self-contained: worker reads only this skill + has the inputs from spawn prompt

**Anti-pattern:** building one skill that mixes both audiences (user workflow + embedded worker prompt template + worker-side technical methodology) → structurally confused, neither side can navigate cleanly. Symptom: skill exceeds ~250 LOC and tries to address both Opus and a hypothetical worker in the same file.

**Rule:** BEFORE writing a skill, answer:
1. Who reads this — Opus (main session, deciding) or Worker (after activation, executing)?
2. If Opus: workflow guide. Decisions, options, user-input gates, optional worker-spawn at the end.
3. If Worker: methodology. Pure execution steps. No spawn template. No "ask user".

If a workflow needs both: TWO skills. Opus-side describes the user workflow + ends with "spawn worker pointing at <X>-skill". Worker-side contains the methodology that gets activated post-spawn.

Concrete failure (2026-05-06, searxng): First attempt at `crawl-and-index/SKILL.md` (224 LOC) embedded a worker-prompt-template AND the 5-shape methodology AND user-facing decisions in one file. User corrected: "nein das macht eben keinen sinn ich will kein template im skill haben. ich will das der worker den skill aktiviert. das heißt wir sollten lieber einen cleanup skill haben der einmal für pdf und einmal für web md gilt … einen skill für uns der alles bis zu dem punkt beschriebt an dem der worker übernehmen würde." Re-built as two skills: `web-research/SKILL.md` extended with "Permanent Capture Workflow" section (Opus-side, shows the short worker-spawn pattern), and `cleanup-and-index/SKILL.md` (Worker-side, the technical methodology activated by the worker after spawn). One wasted iteration (~30 min). Audience-first design from the start would have prevented it.

---

## ~/.claude/shared-rules/opus/beads.md → "Defer pattern" subsection

### No `bd defer` subcommand — close-with-defer-reason

`bd` CLI does NOT expose a `defer` subcommand as of 2026-05-06. The `❄ deferred` status icon shown in `bd list` legend exists in the schema, but no CLI command sets it (probably set programmatically or via direct dolt edits).

**Pattern when a bead should be deferred** (parked, not deleted, may resurface later):

Use `bd close <id> --reason="Parked — <reason>. Pick up after <trigger>. Re-open this bead OR create new focused bead at that point."`. This documents the defer intent in the close-reason for fresh-Opus reading. The bead is technically closed but the reason makes the parked nature explicit.

Concrete failure (2026-05-06): Attempted `bd defer searxng-baw --reason="..."` — bd returned generic help output (not the action). Wasted one tool call. Should have been close-with-defer-reason from the start.

---

## ~/.claude/shared-rules/opus/chat-output.md → New section "Fahrplan Completeness"

### Session-end Fahrplan MUST include dev → main step

When laying out a multi-phase Fahrplan to reach session-closing state, ALWAYS include the explicit `dev → main` sync step. The sync is easy to forget because it sits AFTER all visible work (merge + bead-close + verification). Without it, next session's work runs against stale main.

**Required steps in any session-end Fahrplan that involves dev-branch work:**

1. Worker waits done
2. Merges + plugin-syncs
3. Sanity verification on dev
4. Bead operations (close/defer/comment)
5. **dev → main sync** (`git checkout main && git merge dev`) — often forgotten
6. Optional push

Step 5 is non-negotiable for any session that produced commits on dev. The plugin-syncs and worker merges land code on dev only; main remains stale until step 5.

Concrete failure (2026-05-06, searxng cleanup-and-index): Initial Fahrplan had Phases 1-5 (workers + merge + sanity + bead closes + verify final state). No dev → main step. User caught it: "ok udn prod migration?" Had to acknowledge the gap, add Phase 7 (dev → main) + Phase 8 (push question) explicitly. Fundamental for prod-migration sessions; should be the default Fahrplan template.
