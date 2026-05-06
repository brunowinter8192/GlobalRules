# Verify Before Execution

## Kill Assumptions by Action (CRITICAL)

BEFORE executing any plan: convert each assumption into a concrete verification action — `ls` the path, `grep` the reference, read the file, run a 1-call live test against the API. Do NOT just label an assumption "high confidence" and move on. Labeling is communication (see `communication-2.md`); THIS rule is about the action that kills the assumption before it costs iterations.

**Rule:** Every assumption gets paired with the cheapest concrete action that would falsify it. Run the action BEFORE acting on the assumption.

Concrete failure (2026-03-31): Assumed Claude Code hooks v2.1.89 supports 50KB additionalContext per hook. Source: GitHub CHANGELOG. Confidence: high. Reality: 10KB limit. A 30-second live test would have caught this. 5 iterations wasted designing around a false premise.

## Verify Inputs (Execution)

BEFORE executing: verify inputs exist (paths with `ls`, tool params with `--help`, API shapes via docs). NEVER construct from memory. When running multiple similar commands: execute ONE first, verify, then batch.

**Search/Keyword Recommendations:** ALWAYS verify keyword recommendations with a live test (e.g., MCP search_posts, 1 query per keyword) BEFORE presenting to user. Never recommend keywords based on assumptions about what "should work".
- Concrete failure (2026-03-23): Recommended "KI Agents", "KI Tools", "KI Strategie" as German keywords — all three produced >80% non-German noise. Only discovered after user said "nutze mal selber die mcp tools".

**File Entry Format:** BEFORE grep/count on MD files (batch files, exports, reports): read ONE file first to understand the entry format. Never assume header level or pattern.
- Concrete failure (2026-03-26): Counted `## ` headers in job batch files → 55 jobs. Correct pattern was `N. **` → 156 jobs. Format was visible in the first file — just not read.

**Tabular File Write-In:** BEFORE drafting new row content for a worker prompt or your own append, READ the file to verify column count, separator style, and alignment. Existing-row format is the source of truth — do NOT rely on memory or a generic "decision-files format" / "sources-format" template carried from rules or training. Inspect the file in this session, copy the column structure literally. A meta-instruction "match existing format" in the prompt is NOT enough — when literal text and meta-instruction conflict, models lean literal.
- Concrete failure (2026-05-01, searxng): drafted 8 new rows for `searxng/sources/sources.md` with 5 columns (`Source | Domain | Type | Decisions | Status`). Existing rows had 4 columns (no Domain). Worker followed draft literally despite prompt saying "normalize to existing format". Format mismatch caught only post-merge → worker_send correction round + amended commit. 30 seconds reading sources.md before drafting would have prevented it.

**Shared File Deletion:** BEFORE deleting a shared file (script, config, library): `grep -r <filename> <parent-dir>/` to find all references. Fix or remove references BEFORE deleting the file.
- Concrete failure (2026-03-29): Deleted `install_deps.sh` without checking references → 3 scripts (`compile_latex.sh`, `pdf_to_images.sh`, `latex_lint.sh`) all sourced it → all broken. One grep would have prevented 3 fix iterations.

**Tool Extension:** BEFORE building a new tool/server/repo: READ the existing server.py or codebase where the tool SHOULD live. Adding a function to an existing server is almost always the right answer. Never create a new repo/venv/MCP server for functionality that belongs in an existing one.
- Concrete failure (2026-03-30): Built a standalone llm-proxy MCP server (new repo, new venv, new .mcp.json entry) instead of adding a `prompt()` function to iterative-dev's server.py. User had even said "schau mal wie das tool funktioniert" — pointing at the existing server. 2 corrections needed.

**Numeric Values in Reports / Analysis / Chat:** NEVER state specific numeric values (config settings, limits, counts, token budgets, message counts, prices, rates, latencies, benchmark numbers) from memory when documenting, analyzing, OR speaking in chat. Always verify against the actual data source (raw_payload, file content, config, README, API doc, live measurement) before writing or stating. A single wrong numeric value undermines trust in the entire analysis. Either verify, or explicitly label "from training knowledge, not verified this session" — never present unverified as fact.
- Concrete failure (2026-04-13): Wrote `max_tokens: 32000` in a REQ#1 payload structure example during Monitor_CC forensic investigation. Actual value in the proxy-logged raw_payload was `64000`. User had to correct before continuing. Reading `raw_payload.max_tokens` directly would have taken 2 seconds.
- Concrete failure (2026-04-22, searxng): Claimed "2captcha costs ~$0.003/solve" in chat while discussing paid-captcha architecture. Number came from training knowledge, not verified. User asked "was ist das denn?" — had to admit unverified. A 10-second README grep / web fetch would have surfaced the actual current price or led to "I don't have the 2026 price".
- **Cumulative vs Per-Request format.** When presenting cumulative metrics across a multi-request session (total cache-read tokens, total input chars, total cost), the number MUST be paired with the per-request average and the request count. Never present only the cumulative number. Format: "<total> across <N> requests → avg <total/N> per request". Example: "71.8M cache-read tokens across 321 requests → avg ~224k per request". Raw cumulative numbers can be misinterpreted as unique-token-count or maximum-context-size; per-request framing makes prefix size + multiplier explicit.
- Concrete failure (2026-04-21, Monitor_CC): "71.8M tokens" presented as standalone number in session summary. User immediately challenged "das kann gar nicht sein" because the number read as "unique token volume". After per-request framing (avg 170k per request × 305 requests) the number became self-explanatory.

**Pattern Generalization Across Similar Targets:** BEFORE claiming "same refactor/conversion pattern applies to N repos/modules/files" — inventory EACH target first. Read file counts, module shapes, dependency lists, and a representative internal file per target. A cheap inventory pass (15-30 seconds per target via `ls` + `grep -c` + head-read) prevents retroactive scope reduction after workers are already dispatched on a wrong assumption. Never dispatch workers to N targets on a pattern proven on only ONE.
- Concrete failure (2026-04-14 evening): After successfully converting `MCP/github` from MCP to Skill+CLI (thin-wrapper pattern), Opus immediately claimed "reddit, arxiv, RAG follow the exact same pattern — trivial conversion". User challenged: "naja ist das denn machbar … wissen wir doch gar nicht. ich hatte bei reddit und gh srix einige grenzen enforced". Scope was correctly reduced to "github only first, inventory the others after". A single `ls` + `wc -l server.py` + `grep @mcp.tool` pass across the 3 repos (~30 seconds) would have let Opus make the claim with evidence instead of projection. The claim happened to be true, but was verified only retroactively. Do the inventory FIRST, claim SECOND.

**Registry Before Proxy / Strip Analysis:** When a Skill/Tool invocation fails ("Unknown skill", "No such tool available", "validation error"), check the REGISTRATION SOURCE first (`plugin.json`, `.mcp.json`, `settings.local.json` enabledPlugins, plugin cache dir) BEFORE hypothesizing about proxy mutations, strip rules, system-reminder injection, or other runtime mechanisms. Registry drift is 1 file read to detect; runtime mutation analysis is 10+ tool calls. The cheaper diagnostic goes first.
- Concrete failure (2026-04-15): `Skill(skill="bead-cli")` returned "Unknown skill". Opus immediately theorized that the proxy was stripping the available-skills system-reminder and prevented the Skill tool from discovering bead-cli. Spent 4 tool calls grepping proxy strip logic, reading `rules.py:302`, extracting historical stripped SR content from proxy logs. Root cause was trivial: `~/Documents/ai/Meta/blank/.claude-plugin/plugin.json` declared only 8 skills in its `skills[]` array — bead-cli + 5 others were missing. One `cat plugin.json | jq .skills` call would have ended the investigation. The Skills SR strip was real but irrelevant — even unstripped, the SR listing also relies on plugin.json registration.

**Access-Path Assumption (External Services):** BEFORE proposing a solution path that relies on an external API endpoint, SDK, or service integration, verify the user's access mode:

- API key (direct Anthropic/OpenAI/etc.) — pay-per-use, full endpoint access
- OAuth token (Claude Code on Max/Pro subscription) — endpoint-restricted by Anthropic
- Subscription plan (Max, Pro, Team) — different rate limits and feature gates
- Enterprise / custom deployment

Subscription users CANNOT access all API endpoints. Specifically: Anthropic blocks `/v1/messages/count_tokens` on OAuth tokens (returns canned `400 "max_tokens: Extra inputs are not permitted"` regardless of payload). `/v1/messages` works with both. Other endpoints vary.

When in doubt: live-test with a minimal request (one curl with the user's actual auth) BEFORE committing to the path. A 10-second test saves proposing dead-ends.

Concrete failure (2026-04-17): Proposed `/count_tokens` as ground-truth path for tokenizer baseline without confirming user's access mode. User was on Max plan (OAuth only). Live test revealed blocked endpoint → 1-2 exchanges wasted on a path that never could have worked.

**GitHub Repo Maintenance Claims:** before stating a repo is "actively maintained" or "current state-of-the-art" in chat or reports, call `list_commits owner repo --per-page 1` (or equivalent) and read the `Date:` field. Do NOT rely on `search_repos` output `Updated: YYYY-MM-DD` — that field reflects GitHub's indexing/push events including Dependabot bumps, README-only edits, and metadata touches, and routinely shows "recent" dates for repos whose last substantial commit is years old.
- Concrete failure (2026-04-22, searxng): yagooglesearch presented as "aktiv gepflegt" based on `search_repos` Updated 2026-04-09. `list_commits` revealed last commit 2024-04-07 (2 years stale). Re-ranking after correction surfaced `karust/openserp` as the real current state-of-art (10+ commits in April 2026).

**Venv Path Naming Varies by Project:** some projects use `.venv/`, others `venv/`. NEVER `source .venv/bin/activate` or invoke `.venv/bin/python` from memory across projects — verify with `ls -d <project>/{venv,.venv} 2>/dev/null` or read the project's setup notes first.
- Concrete failure (2026-05-04, Trading session): `cd MCP/RAG && source .venv/bin/activate` failed with "no such file or directory" — RAG uses `venv/`, Trading uses `.venv/`. They live one directory apart on disk; convention is not consistent.

**Tool-Name Memory After Mid-Session Migration:** when a tool is removed/renamed/migrated within an active session (MCP tool replaced with CLI wrapper, schema changed, etc.), the OLD tool name remains in Opus's already-loaded system prompt for the rest of that session. Any invocation using the old name fails with "Error: No such tool available" — tool discovery happens at session start and is frozen. Use the replacement directly for any subsequent call.
- Concrete failure (2026-04-21, Monitor_CC cli-consolidation): `worker_merge` MCP was removed and replaced with `worker-cli merge` mid-session. Later in the same session, Opus attempted `mcp__plugin_iterative-dev_iterative-dev__worker_merge` — got "No such tool available" error, wasted input tokens for zero output.

## Root Cause Before Fix

NEVER implement defensive fixes (validation, silent skip, fallback) without understanding the root cause. Symptom treatment masks the real problem. If root cause is unclear after investigation: inform user explicitly, don't silently add guards.

## Leverage-Claim Verification (MANDATORY)

When proposing an intervention based on a %-leverage claim against a defined dataset (e.g. "this rule catches 50% of the problem cases", "this hook blocks 30% of the retry loops"):

BEFORE user approves the plan, walk EVERY case in the dataset individually and mark whether the intervention would actually address it. Present the case-by-case mapping in the plan, not the aggregate number alone.

Aggregate categories lie. "50% of zeros are path-not-found" is NOT the same as "a path-preflight hook catches 50% of zeros" — because path-not-found includes sub-cases with no checkable prefix, cases already handled by native tool errors, and cases where the error message would not change behavior.

Test before claim: "Can I produce a table with case-id / would-lever-catch: yes|no?" If no → the leverage claim is speculative, label as hypothesis not solution.

Concrete failure (2026-04-19): Claimed 50% leverage for path-preflight PreToolUse hook on 21 zero-result cases (eew bead). Actual per-case walk: 2/21. User approved plan based on 50%, worker built hook + monitor visualization, live-verified, only on user's question "welchen Mehrwert soll das denn bringen?" did the per-case walk happen → rollback required (2 git reverts, ~/.claude/hooks deletion, settings restore). The per-case check must happen UP FRONT, not retroactively under pushback.

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

Concrete failure (2026-05-01, Monitor_CC): drafted Immutability rule as "prefer pure functions, return new objects; mutation only when explicitly justified". Sounded right. AST scan against `src/` would have found 503 `.append()` calls on locals, 44 `.extend()`, 6 `.update()` — most idiomatic line-building in render functions. Codifying the broad version would have flagged 552 false positives. The reality-check (user-requested via "check das mal gegen die codebase monitor") narrowed the rule to "Functions must not mutate their arguments" — 49 hits in same codebase, all true positives matching the actual LLM anti-pattern. 10× FP-ratio gap, fixed in 2 minutes via AST scan before commit.

## Correlation Check Before Root-Cause Claim

When proposing a hypothesis as root cause based on ONE observed event, scan the SAME dataset for OTHER instances matching the same precondition pattern. If the precondition occurs frequently but only ONE instance had the symptom → hypothesis is INSUFFICIENT. Present as "necessary but not sufficient" — never as root cause.

Concrete failure (2026-04-12): Presented "shape demotion → cache rebuild" as root cause for monitor_cc REQ#33 after finding a single shape-demotion between REQ#32 and REQ#33. User asked "können wir reproduzieren?" — scan of the same log showed 11 shape-demotions in the session, only 1 with a catastrophic rebuild. The hypothesis was a contributing factor, not the cause. A 30-second precondition-scan before claiming root cause would have prevented the overclaim.

**Cross-Session / Cross-Case Falsification.** When investigating a specific rebuild/bug/anomaly event, the correlation check above covers the SAME session. Cross-session is the next layer: BEFORE labeling any cause as root cause, scan OTHER sessions/logs/cases with the same observable precondition pattern. If the precondition exists in cases WITHOUT the symptom → hypothesis is insufficient. Label as "contributing factor" at best. Do NOT commit to a fix based on a hypothesis that hasn't been counter-example tested across available historical data.

Concrete failure (2026-04-13): During Monitor_CC REQ#23→#24 double-rebuild investigation, presented "Write tool's `]`→`, {` byte transition breaks cache" as root cause after finding exactly that byte-level change in the current session payload. A 60-second scan of the previous session's proxy log showed REQ#33→#34 had the IDENTICAL byte transition (tools 10→16, same mcp__worker_* defer_loading tools, `]`→`, {"` after Write's cc_marker) with the OPPOSITE outcome (CR recovery 36k→104k instead of catastrophic rebuild). Hypothesis was falsified. Four separate hypotheses were proposed and falsified in sequence during the same investigation because each was presented as near-conclusion before cross-session counter-example checks.

**Investigation-Context Contamination.** When investigating a historical claim ("X was removed in version Y", "feature Z disappeared after date D") and finding what looks like counter-evidence in recent files (today's logs, current session, last 24h), STOP before drawing the conclusion. Ask: is the recent file part of the investigation context itself? A grep/scan that finds the disputed text in a session whose stated purpose was to verify that exact text is NOT counter-evidence — the text appears because the investigation put it there (quoting in tool prompts, reading drafts that contain it, pasting into messages). Before claiming "the claim is wrong, X is still happening": cross-reference the file's mtime / session purpose against the topic. Single-day clusters in otherwise-clean historical data are a red flag for self-contamination, not a counter-example.

Concrete failure (2026-04-30, Monitor_CC LinkedIn-claim verification): verified post claim that a malware-themed system-reminder was removed in CC 2.1.109. Found 105 occurrences of the exact reminder text ("Whenever you read a file, you should consider whether it would be considered malware...") in proxy log of CC 2.1.114 from 2026-04-29. Hypothesis presented as "reminder returned in 2.1.114". Wrong. 2026-04-29 was the verification session in which Opus and user had read post drafts containing the quoted reminder text and grepped logs containing it. User: "ja weil wir es gestern auch verifiziert haben — sollte es dir nicht zu denken geben dass ausschließlich gestern die reminder auftraten nach dem cutoff?". The single-date cluster was the contamination signal, ignored.

**Systematic Audit on User Counter-Evidence.** When user provides a live screenshot / concrete instance contradicting Opus's claim that "X is handled / stripped / filtered / fixed":

1. Do NOT fix the single visible case first.
2. FIRST write a batch audit (script against the same data source that produced the screenshot) that enumerates ALL instances of the pattern, classified by relevant dimensions (type × location × session).
3. THEN decide whether it's one broken case or a systemic gap. Often it's systemic.

Rule: user's counter-evidence = audit flag, not one-off fix trigger.

Concrete failure (2026-04-19 evening): user showed an unstripped `<system-reminder>` in the proxy display. First instinct was to explain "maybe proxy shows pre-strip for transparency" + a single-case raw_payload grep. After user insisted ("ich hab es satt ständig system reminder zu finden"), Opus wrote a systematic audit and found 24,282 bypassed SRs across 6 recent logs — systemic, not single-case. The strip functions didn't recurse into `tool_result.content` strings.
