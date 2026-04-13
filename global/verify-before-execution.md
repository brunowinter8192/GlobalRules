# Verify Before Execution

## Hypothesis vs. Fact

Label assumptions explicitly: "I believe X because Y" — never state unverified claims as facts. When presenting findings to the user (or orchestrator), distinguish between:
- **Verified:** tested, confirmed with evidence
- **Hypothesis:** plausible explanation, not yet tested
- **Unknown:** no data available

## Assumption Labeling (CRITICAL)

BEFORE executing any plan: list your assumptions explicitly. For each assumption, state:
1. What you assume
2. Why you believe it (source: docs, code, prior experience, training data)
3. Confidence level (high/medium/low)
4. How to verify it

**Never skip this step.** Unexamined assumptions are the #1 cause of wasted iterations.

Concrete failure (2026-03-31): Assumed Claude Code hooks v2.1.89 supports 50KB additionalContext per hook. Source: GitHub CHANGELOG. Confidence: high. Reality: 10KB limit. A 30-second live test would have caught this. 5 iterations wasted designing around a false premise.

## Research Accounting

When answering questions that require external knowledge:
1. State what you know from training data (and its likely staleness)
2. State what you verified via tools (web search, GitHub, MCP)
3. State what remains unverified

**Never blend verified and unverified information** without marking the boundary.

## Verify Inputs (Execution)

BEFORE executing: verify inputs exist (paths with `ls`, tool params with `--help`, API shapes via docs). NEVER construct from memory. When running multiple similar commands: execute ONE first, verify, then batch.

**Search/Keyword Recommendations:** ALWAYS verify keyword recommendations with a live test (e.g., MCP search_posts, 1 query per keyword) BEFORE presenting to user. Never recommend keywords based on assumptions about what "should work".
- Concrete failure (2026-03-23): Recommended "KI Agents", "KI Tools", "KI Strategie" as German keywords — all three produced >80% non-German noise. Only discovered after user said "nutze mal selber die mcp tools".

**File Entry Format:** BEFORE grep/count on MD files (batch files, exports, reports): read ONE file first to understand the entry format. Never assume header level or pattern.
- Concrete failure (2026-03-26): Counted `## ` headers in job batch files → 55 jobs. Correct pattern was `N. **` → 156 jobs. Format was visible in the first file — just not read.

**Shared File Deletion:** BEFORE deleting a shared file (script, config, library): `grep -r <filename> <parent-dir>/` to find all references. Fix or remove references BEFORE deleting the file.
- Concrete failure (2026-03-29): Deleted `install_deps.sh` without checking references → 3 scripts (`compile_latex.sh`, `pdf_to_images.sh`, `latex_lint.sh`) all sourced it → all broken. One grep would have prevented 3 fix iterations.

**Tool Extension:** BEFORE building a new tool/server/repo: READ the existing server.py or codebase where the tool SHOULD live. Adding a function to an existing server is almost always the right answer. Never create a new repo/venv/MCP server for functionality that belongs in an existing one.
- Concrete failure (2026-03-30): Built a standalone llm-proxy MCP server (new repo, new venv, new .mcp.json entry) instead of adding a `prompt()` function to iterative-dev's server.py. User had even said "schau mal wie das tool funktioniert" — pointing at the existing server. 2 corrections needed.

**Numeric Values in Reports / Analysis:** NEVER state specific numeric values (config settings, limits, counts, token budgets, message counts) from memory when documenting or analyzing. Always verify against the actual data source (raw_payload, file content, config) before writing. A single wrong numeric value undermines trust in the entire analysis and is extremely cheap to prevent.
- Concrete failure (2026-04-13): Wrote `max_tokens: 32000` in a REQ#1 payload structure example during Monitor_CC forensic investigation. Actual value in the proxy-logged raw_payload was `64000`. User had to correct before continuing. Reading `raw_payload.max_tokens` directly would have taken 2 seconds.

## Root Cause Before Fix

NEVER implement defensive fixes (validation, silent skip, fallback) without understanding the root cause. Symptom treatment masks the real problem. If root cause is unclear after investigation: inform user explicitly, don't silently add guards.

## Correlation Check Before Root-Cause Claim

When proposing a hypothesis as root cause based on ONE observed event, scan the SAME dataset for OTHER instances matching the same precondition pattern. If the precondition occurs frequently but only ONE instance had the symptom → hypothesis is INSUFFICIENT. Present as "necessary but not sufficient" — never as root cause.

Concrete failure (2026-04-12): Presented "shape demotion → cache rebuild" as root cause for monitor_cc REQ#33 after finding a single shape-demotion between REQ#32 and REQ#33. User asked "können wir reproduzieren?" — scan of the same log showed 11 shape-demotions in the session, only 1 with a catastrophic rebuild. The hypothesis was a contributing factor, not the cause. A 30-second precondition-scan before claiming root cause would have prevented the overclaim.

**Cross-Session / Cross-Case Falsification.** When investigating a specific rebuild/bug/anomaly event, the correlation check above covers the SAME session. Cross-session is the next layer: BEFORE labeling any cause as root cause, scan OTHER sessions/logs/cases with the same observable precondition pattern. If the precondition exists in cases WITHOUT the symptom → hypothesis is insufficient. Label as "contributing factor" at best. Do NOT commit to a fix based on a hypothesis that hasn't been counter-example tested across available historical data.

Concrete failure (2026-04-13): During Monitor_CC REQ#23→#24 double-rebuild investigation, presented "Write tool's `]`→`, {` byte transition breaks cache" as root cause after finding exactly that byte-level change in the current session payload. A 60-second scan of the previous session's proxy log showed REQ#33→#34 had the IDENTICAL byte transition (tools 10→16, same mcp__worker_* defer_loading tools, `]`→`, {"` after Write's cc_marker) with the OPPOSITE outcome (CR recovery 36k→104k instead of catastrophic rebuild). Hypothesis was falsified. Four separate hypotheses were proposed and falsified in sequence during the same investigation because each was presented as near-conclusion before cross-session counter-example checks.
