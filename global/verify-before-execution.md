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

## Root Cause Before Fix

NEVER implement defensive fixes (validation, silent skip, fallback) without understanding the root cause. Symptom treatment masks the real problem. If root cause is unclear after investigation: inform user explicitly, don't silently add guards.
