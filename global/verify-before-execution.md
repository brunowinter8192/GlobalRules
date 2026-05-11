# Verify Before Execution

## Kill Assumptions by Action (CRITICAL)

BEFORE executing any plan: convert each assumption into a concrete verification action — `ls` the path, `grep` the reference, read the file, run a 1-call live test against the API. Do NOT just label an assumption "high confidence" and move on. Labeling is communication (see `communication-2.md`); THIS rule is about the action that kills the assumption before it costs iterations.

**Rule:** Every assumption gets paired with the cheapest concrete action that would falsify it. Run the action BEFORE acting on the assumption.


## Verify Inputs (Execution)

BEFORE executing: verify inputs exist (paths with `ls`, tool params with `--help`, API shapes via docs). NEVER construct from memory. When running multiple similar commands: execute ONE first, verify, then batch.

**Search/Keyword Recommendations:** ALWAYS verify keyword recommendations with a live test (e.g., MCP search_posts, 1 query per keyword) BEFORE presenting to user. Never recommend keywords based on assumptions about what "should work".

**File Entry Format:** BEFORE grep/count on MD files (batch files, exports, reports): read ONE file first to understand the entry format. Never assume header level or pattern.

**Tabular File Write-In:** BEFORE drafting new row content for a worker prompt or your own append, READ the file to verify column count, separator style, and alignment. Existing-row format is the source of truth — do NOT rely on memory or a generic "decision-files format" / "sources-format" template carried from rules or training. Inspect the file in this session, copy the column structure literally. A meta-instruction "match existing format" in the prompt is NOT enough — when literal text and meta-instruction conflict, models lean literal.

**Shared File Deletion:** BEFORE deleting a shared file (script, config, library): `grep -r <filename> <parent-dir>/` to find all references. Fix or remove references BEFORE deleting the file.

**Tool Extension:** BEFORE building a new tool/server/repo: READ the existing server.py or codebase where the tool SHOULD live. Adding a function to an existing server is almost always the right answer. Never create a new repo/venv/MCP server for functionality that belongs in an existing one.

**Numeric Values in Reports / Analysis / Chat:** NEVER state specific numeric values (config settings, limits, counts, token budgets, message counts, prices, rates, latencies, benchmark numbers) from memory when documenting, analyzing, OR speaking in chat. Always verify against the actual data source (raw_payload, file content, config, README, API doc, live measurement) before writing or stating. A single wrong numeric value undermines trust in the entire analysis. Either verify, or explicitly label "from training knowledge, not verified this session" — never present unverified as fact.
- **Cumulative vs Per-Request format.** When presenting cumulative metrics across a multi-request session (total cache-read tokens, total input chars, total cost), the number MUST be paired with the per-request average and the request count. Never present only the cumulative number. Format: "<total> across <N> requests → avg <total/N> per request". Example: "71.8M cache-read tokens across 321 requests → avg ~224k per request". Raw cumulative numbers can be misinterpreted as unique-token-count or maximum-context-size; per-request framing makes prefix size + multiplier explicit.

**Pattern Generalization Across Similar Targets:** BEFORE claiming "same refactor/conversion pattern applies to N repos/modules/files" — inventory EACH target first. Read file counts, module shapes, dependency lists, and a representative internal file per target. A cheap inventory pass (15-30 seconds per target via `ls` + `grep -c` + head-read) prevents retroactive scope reduction after workers are already dispatched on a wrong assumption. Never dispatch workers to N targets on a pattern proven on only ONE.

**Registry Before Proxy / Strip Analysis:** When a Skill/Tool invocation fails ("Unknown skill", "No such tool available", "validation error"), check the REGISTRATION SOURCE first (`plugin.json`, `.mcp.json`, `settings.local.json` enabledPlugins, plugin cache dir) BEFORE hypothesizing about proxy mutations, strip rules, system-reminder injection, or other runtime mechanisms. Registry drift is 1 file read to detect; runtime mutation analysis is 10+ tool calls. The cheaper diagnostic goes first.

**Access-Path Assumption (External Services):** BEFORE proposing a solution path that relies on an external API endpoint, SDK, or service integration, verify the user's access mode:

- API key (direct Anthropic/OpenAI/etc.) — pay-per-use, full endpoint access
- OAuth token (Claude Code on Max/Pro subscription) — endpoint-restricted by Anthropic
- Subscription plan (Max, Pro, Team) — different rate limits and feature gates
- Enterprise / custom deployment

Subscription users CANNOT access all API endpoints. Specifically: Anthropic blocks `/v1/messages/count_tokens` on OAuth tokens (returns canned `400 "max_tokens: Extra inputs are not permitted"` regardless of payload). `/v1/messages` works with both. Other endpoints vary.

When in doubt: live-test with a minimal request (one curl with the user's actual auth) BEFORE committing to the path. A 10-second test saves proposing dead-ends.


**GitHub Repo Maintenance Claims:** before stating a repo is "actively maintained" or "current state-of-the-art" in chat or reports, call `list_commits owner repo --per-page 1` (or equivalent) and read the `Date:` field. Do NOT rely on `search_repos` output `Updated: YYYY-MM-DD` — that field reflects GitHub's indexing/push events including Dependabot bumps, README-only edits, and metadata touches, and routinely shows "recent" dates for repos whose last substantial commit is years old.

**Venv Path Naming Varies by Project:** some projects use `.venv/`, others `venv/`. NEVER `source .venv/bin/activate` or invoke `.venv/bin/python` from memory across projects — verify with `ls -d <project>/{venv,.venv} 2>/dev/null` or read the project's setup notes first.
**Tool-Name Memory After Mid-Session Migration:** when a tool is removed/renamed/migrated within an active session (MCP tool replaced with CLI wrapper, schema changed, etc.), the OLD tool name remains in Opus's already-loaded system prompt for the rest of that session. Any invocation using the old name fails with "Error: No such tool available" — tool discovery happens at session start and is frozen. Use the replacement directly for any subsequent call.
## Root Cause Before Fix

NEVER implement defensive fixes (validation, silent skip, fallback) without understanding the root cause. Symptom treatment masks the real problem. If root cause is unclear after investigation: inform user explicitly, don't silently add guards.

## Leverage-Claim Verification (MANDATORY)

When proposing an intervention based on a %-leverage claim against a defined dataset (e.g. "this rule catches 50% of the problem cases", "this hook blocks 30% of the retry loops"):

BEFORE user approves the plan, walk EVERY case in the dataset individually and mark whether the intervention would actually address it. Present the case-by-case mapping in the plan, not the aggregate number alone.

Aggregate categories lie. "50% of zeros are path-not-found" is NOT the same as "a path-preflight hook catches 50% of zeros" — because path-not-found includes sub-cases with no checkable prefix, cases already handled by native tool errors, and cases where the error message would not change behavior.

Test before claim: "Can I produce a table with case-id / would-lever-catch: yes|no?" If no → the leverage claim is speculative, label as hypothesis not solution.

## Rule Formulation Reality-Check (BEFORE Codifying)

When proposing a NEW universal rule (worker rule, project rule, global behavior rule), validate the rule's formulation against ≥1 actual codebase BEFORE writing it to the rule file. The reality-check is a 30-60 second AST-grep-or-find pass producing a hit count + sample of false positives. The hit count tells you whether the rule produces signal or noise.

A rule formulation that sounds clean in prose can fire on hundreds of legitimate cases when applied to real code. Codifying the prose-clean version creates a rule that workers must defend against constantly, generates noise in audits, and undermines trust in all rules. Tightening from "prefer pure functions, never mutate" to "Functions must not mutate their arguments" looks subtle in prose but has a 10× false-positive ratio gap on real Python (503 idiomatic `lines.append()` line-builds vs 49 actual argument-mutations).

Process:
1. State the rule formulation in prose.
2. Pick the cheapest tool that approximates the rule (AST script, grep, find).
3. Run on an actual codebase with a representative shape.
4. Count hits. Inspect the first 10-20.
5. Classify: TRUE positive (rule should flag) vs FALSE positive (rule formulation too broad).
6. If FP ratio > 10% → rule formulation is wrong. Tighten until below 10%, OR add explicit carve-out language.
7. Codify the validated formulation.

Applies to: any addition to `~/.claude/shared-rules/worker/`, `~/.claude/shared-rules/global/`, project-specific rule files. Does NOT apply to `decisions/` (project-specific IST notes, not behavior rules), tool-use shortcuts (factual constraints), or one-off session rules.

## Correlation Check Before Root-Cause Claim

When proposing a hypothesis as root cause based on ONE observed event, scan the SAME dataset for OTHER instances matching the same precondition pattern. If the precondition occurs frequently but only ONE instance had the symptom → hypothesis is INSUFFICIENT. Present as "necessary but not sufficient" — never as root cause.

**Cross-Session / Cross-Case Falsification.** When investigating a specific rebuild/bug/anomaly event, the correlation check above covers the SAME session. Cross-session is the next layer: BEFORE labeling any cause as root cause, scan OTHER sessions/logs/cases with the same observable precondition pattern. If the precondition exists in cases WITHOUT the symptom → hypothesis is insufficient. Label as "contributing factor" at best. Do NOT commit to a fix based on a hypothesis that hasn't been counter-example tested across available historical data.

**Investigation-Context Contamination.** When investigating a historical claim ("X was removed in version Y", "feature Z disappeared after date D") and finding what looks like counter-evidence in recent files (today's logs, current session, last 24h), STOP before drawing the conclusion. Ask: is the recent file part of the investigation context itself? A grep/scan that finds the disputed text in a session whose stated purpose was to verify that exact text is NOT counter-evidence — the text appears because the investigation put it there (quoting in tool prompts, reading drafts that contain it, pasting into messages). Before claiming "the claim is wrong, X is still happening": cross-reference the file's mtime / session purpose against the topic. Single-day clusters in otherwise-clean historical data are a red flag for self-contamination, not a counter-example.

**Systematic Audit on User Counter-Evidence.** When user provides a live screenshot / concrete instance contradicting Opus's claim that "X is handled / stripped / filtered / fixed":

1. Do NOT fix the single visible case first.
2. FIRST write a batch audit (script against the same data source that produced the screenshot) that enumerates ALL instances of the pattern, classified by relevant dimensions (type × location × session).
3. THEN decide whether it's one broken case or a systemic gap. Often it's systemic.

Rule: user's counter-evidence = audit flag, not one-off fix trigger.
