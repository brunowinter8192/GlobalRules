# Documentation Hierarchy

Documentation lives in two parallel chains.

**External-facing chain** (human users finding the project on GitHub):
- README.md — entry-point + setup + features for someone external

**AI-internal chain** (granularity gradient for AI exploration, coarse → fine):
- DOCS.md (root) — project entry-point: pipeline components / key files / documentation tree to subdir DOCS.md.
- decisions/*.md — per-pipeline-step rationale, FINAL STATE ONLY (prose-level: IST/Evidenz/SOLL/Quellen). Format spec: see `## decisions/` below.
- decisions/OldThemes/<topic>/ — process documentation: iterations, alternatives explored, superseded values, dead ends. Format spec: see `### OldThemes` below.
- src/<package>/DOCS.md — per-module structured map (Role/Modules/LOC/Called-by)
- source code — full detail
- dev/*.md — investigations, probes, evals (sit beside the chain, not inside)

## Artifact Density

User-Chat is prose. Everything Claude reads or produces as ARTIFACT — code, DOCS.md, rules/, decisions/, code-comments — is machine-readable and token-dense.

Concrete: tables instead of prose where multiple dimensions need comparison, keywords instead of full sentences, references to `file:line` instead of explanatory paragraphs, no rhetorical filler ("furthermore", "as we can see", "importantly", "it's worth noting"). Where a paragraph IS needed, it is dense — no repetition, no opening sentences that say nothing.

Applies to every project, not only Monitor_CC.

## No Issue References in Docs

`decisions/*.md`, `decisions/OldThemes/**/*.md`, `**/DOCS.md` NEVER reference issues. The direction is one-way: issues point to docs, docs don't point back.

## RAG Collection Layers

Per project: **two** logical collections.

1. **docs** — alle internen Projekt-Dokumente: `DOCS.md`, `decisions/*.md`, `decisions/OldThemes/**`
2. **reference** — alle externen Quellen: vendor docs (z.B. Anthropic API docs), Papers, third-party Repos

**Canonical naming:**

| Layer | Convention | Example |
|---|---|---|
| docs | `<Project>-docs` | `monitor-cc-docs` |
| reference | `<Project>-reference` | `monitor-cc-reference` |

**Use-case routing:**

- Question about own project (code, architecture, decisions, past iterations) → **docs**
- Question about external behavior (API features, framework conventions, library semantics, vendor docs) → **reference**

Search-tool: `~/.claude/shared-rules/global/tool-use.md` § RAG CLI. Gate enforcement: `~/.claude/shared-rules/opus/workers-1.md` § RAG-First on Any Project Question.

**RAG-related OldThemes live in both Monitor_CC and RAG repos.** Files in `decisions/OldThemes/` that concern the RAG system itself are physically duplicated across the two repos and indexed independently by each project's `update_docs`. On edit in either repo: run `sync-rag-oldthemes <filename>` to copy to the other side (mtime-based, picks newer).

## decisions/

Pipeline decision records — FINAL STATE ONLY. Each file documents the current production choice (IST), the evidence backing it (Evidenz), and the recommendation for change if any (SOLL). The process that led to the current choice — alternatives evaluated, superseded values, iteration history — lives in `decisions/OldThemes/<topic>/`, NOT here. A decision file is the crystallized conclusion; OldThemes is the working memory that produced it.

Rule: any value or comparison that is NOT the current production state belongs in OldThemes, even if it was true last week. decisions/ files do not carry historical alternatives.

**IST update timing.** IST in `decisions/<step>.md` reflects current production state. When code changes functionally, IST MUST be updated in the SAME commit cycle as the code change — never deferred. Code-changed-without-IST-update at Recap = drift, blocks session close. Pure refactors with no behavior change update `<package>/DOCS.md` (module shape) but do not touch `decisions/<step>.md` unless a specific path mention becomes stale.

**SOLL → IST direction.** IST changes follow SOLL. There is no "IST functional change without preceding SOLL change". Workflow: (1) discuss in session, alternatives go to OldThemes/, (2) SOLL emerges with dev/ evidence and lands in `decisions/<step>.md` SOLL + Evidenz, (3) migrate IST → match SOLL in the same cycle (code + IST update together). Skipping (2) = process violation; the eval/verification must precede the code change.

### Structure (MANDATORY)

Every decision file has these sections in order:

#### Status Quo (IST)
What is CURRENTLY in production code. Code paths, config values, behavior. This section describes reality — not what should be.

#### Evidenz
Measurements from dev/ scripts, external research, benchmarks. Data that informs the recommendation.

**Internal eval results — MUST cite script AND report-MD AND dataset.** When a decision is justified by a measurement run, the Evidenz section contains the actual numbers (sweep table, baseline values, summary) inline — not just a link. For each cited measurement:
- The dev/ script that produced it (e.g. `dev/retrieval/A_retrieval_eval.py`)
- The report-MD path (e.g. `dev/retrieval/A_retrieval_eval_reports/baseline_2026-04-28.md`)
- The dataset / collection / sample size (e.g. `test_db` on `rag_test`, 250 chunks, 17 queries)

Result-MDs under `dev/<area>/<script>_reports/` are PRIMARY EVIDENCE — they MUST be lifted into the relevant `decisions/<step>.md` Evidenz in the same session as the eval run. Numbers living only in the report artifact = the canonical IST/Evidenz/SOLL record is incomplete and RAG-search on `<Project>-docs` cannot find them.

External evidence (papers, benchmarks from outside research) cites collection + document name from RAG (e.g. `rag-cli-reference: Fusion_Functions_Hybrid_Retrieval`).

#### Recommendation (SOLL)
What the config SHOULD be based on evidence. Three formats:

- **Change:** `X → Y` — Current config should change. State the concrete new value and why.
- **Keep:** `X (no change needed)` — Current config is optimal. One line is enough.
- **Pending:** `Evaluation not done` — No recommendation yet. State what eval is needed. One line is enough.

**Brevity rule:** When SOLL = IST (Keep) or SOLL is unknown (Pending), keep it short — one line per point. Only Change items need detailed justification. This keeps decision files scannable.

When IST ≠ SOLL → migration needed. The diff across ALL decision files = the complete migration plan.

#### Offene Fragen
Unanswered questions that affect the recommendation.

#### Quellen
RAG Collection references, papers, URLs used as evidence. External sources only — internal eval reports belong in Evidenz, not here.

### Path & Symbol References

When citing code in decisions/, OldThemes/, or DOCS.md, the **symbol** (function/constant/class name) is the durable anchor — paths are navigation hints. When code moves, the symbol stays; only the path needs an update.

**Preferred form** (symbol primary, path in parens):
```
`embed_query()` (defined in `src/rag/search_primitives.py`, called via `retriever.py` imports)
`CC_ALPHA = 0.8` in `src/rag/fusion.py`
```

**Discouraged form** (path is the anchor, drifts when code moves):
```
Code: src/rag/retriever.py:embed_query()
```

**When path is the anchor (legitimate, do not symbol-ize):**
- dev/ script paths in Evidenz (the path IS the artifact: `dev/retrieval/A_retrieval_eval.py`)
- Report-MD references in Evidenz (the path IS the citation: `dev/retrieval/A_retrieval_eval_reports/baseline_2026-04-28.md`)
- Decision-file cross-references (`see decisions/box_architecture.md`)
- DOCS.md module listings (the file IS the documented unit, e.g., `### server_manager.py (1061 LOC)`)
- Data files (`dev/retrieval/queries_test_db.json`)

In these cases both the path and any cited content must resolve at Recap.

### Rules

- **IST reflects production code** — not experiments, not plans, not "what we want"
- **SOLL is evidence-based** — every recommendation cites specific eval results from dev/
- **SOLL emergence and changes require dev/ verification** — A SOLL section (Change/Keep/Pending) is added OR modified only when supported by concrete numbers from a dev/ eval, probe, or measurement run THIS session OR explicitly cited from a past report-MD in Evidenz. SOLL changes from opinion/discussion alone are FORBIDDEN — discussion belongs in `decisions/OldThemes/<topic>/` until evidence emerges. Files without eval get a minimal SOLL stub: `## Recommendation (SOLL)\nPending — needs evaluation.`
- **Migration is deferred** — decision files document WHAT should change, not WHEN. All migrations execute in one batch after complete pipeline eval.
- **Eval results propagate to decisions/ in the same session** — when a sweep / baseline / benchmark run produces numbers that inform a pipeline choice, the relevant `decisions/<step>.md` Evidenz section is updated with: numbers inline + dev/ script path + report-MD path + dataset reference. Reports living only in `dev/<area>/<script>_reports/` without a decision-file uplift = drift, caught at Recap.
- **Evidenz from sources only** — Evidenz tables and comparisons MUST come from actually read sources (RAG search results, file reads, GitHub API). NEVER present data from training knowledge as Evidenz. If you haven't read the source in this session, it's an ASSUMPTION, not Evidenz.

### OldThemes

`decisions/OldThemes/<topic>/` is process documentation. It tracks how a decision was reached: alternatives evaluated, superseded values, dead ends, iteration history, open questions per iteration. The corresponding `decisions/<step>.md` only carries the final crystallized choice — OldThemes is the working memory that produced it.

**Purpose split:**

| File | Content | Scope |
|---|---|---|
| `decisions/<step>.md` | Current prod IST + Evidenz that backs it + SOLL recommendation | Final state only |
| `decisions/OldThemes/<topic>/...` | Process: what was tried, what was rejected, what's pending, why current SOLL was chosen | Full history |

**Values in OldThemes ≠ current status quo.** A number measured in iteration 3 is preserved in OldThemes as historical record, not as IST. Only what is currently in production code lives in `decisions/<step>.md`.

**SOLL derives from OldThemes progress.** A decision file's SOLL (Change / Keep / Pending) is the conclusion of the process documented in the matching OldThemes folder. If OldThemes has not converged yet, SOLL stays Pending. If OldThemes shows convergence, SOLL crystallizes the chosen value and `decisions/<step>.md` cites the matching OldThemes file in Evidenz.

**Demotion rule.** When new evidence supersedes an older measurement (new collection, new methodology, new code), the older measurement moves from `decisions/<step>.md` Evidenz to `decisions/OldThemes/<topic>/`. decisions/ never carries stale evidence; OldThemes preserves it for attribution and traceability.

**Naming:**
- Root-level files `decisions/OldThemes/<topic>.md` — single-file themes (e.g. `connection_hang_cascade.md`, `infra03_dynamic_ports.md`, `null_embedding_qwen3_prefix.md`).
- Subfolder `decisions/OldThemes/<topic>/` — multi-file themes. Files inside have free naming (date-based, purpose-based, both OK).

**Subfolder-Trigger:** when a topic grows beyond a single file → create a subfolder, move existing file in, rename if needed.
## DOCS.md

**Audience:** Developer (human or AI).
**Purpose:** Module documentation on the level it lives on.

### Placement Rules

1. **Directory with multiple modules (.py/.sh files)** → DOCS.md lives IN that directory
2. **Directory with single module** → Documented in PARENT directory's DOCS.md (no own DOCS.md)
3. **Hub directories** (directories that only contain subdirectories, no direct modules) → DOCS.md with purpose description + Documentation Tree only
4. **Tightly coupled submodules** (e.g., `retrievers/` inside `eval/`) → Documented in parent DOCS.md, no own DOCS.md. Applies when the subdir is a package of the parent module, not an independent suite.

### Documentation Tree (when sub-DOCS exist)

DOCS.md files that have darunterliegende DOCS MUST include a tree section mapping to them:

```markdown
## Documentation Tree

- [path/to/DOCS.md](path/to/DOCS.md) — One-line description
```

**Rules:**
- Only map the NEXT level of DOCS, not deeper levels
- If a subdirectory has no own DOCS.md (single-file, documented in current DOCS), it does NOT appear in the tree
- The tree is the navigation structure for AI and developers

### DOCS.md Format (Standard)

Purpose: when read FIRST at the start of an exploration, the DOCS.md should let Claude answer in ~30 seconds: Is this my problem? Which modules are affected? Is any module dead or too large? Where are the landmines?

#### Subdir DOCS.md Structure

```markdown
# src/<package>/

## Role
One paragraph — what this package does in the bigger picture, when to touch it, when NOT to touch it.

## Public Interface
What `__init__.py` exports. One line per export. If `__init__.py` is empty: say so and state the actual entry path (e.g. "loaded via `mitmproxy -s`").

## Flow (if relevant)
3-5 lines: data in → processing → data out. Skip for pure-utility packages.

## Modules

### <module>.py (<LOC> LOC)

**Purpose:** one sentence.
**Reads:** data sources (shared state, files, stdin).
**Writes:** outputs (stdout, files, shared state, mutated state).
**Called by:** list of files/packages. Empty list = DEAD CODE, flag explicitly.
**Calls out:** external package dependencies (not stdlib, not `constants`/`utils`).

---

## State (only if package has module-level mutable state)
Which module owns the state, who mutates, who reads.

## Gotchas (optional)
Module-specific landmines. Direct text. No rule-link references (rules are always invoked).
```

#### Root DOCS.md (e.g. `src/DOCS.md`)

```markdown
# src/ — <Project>

## Role
One paragraph: what this src tree does.

## Entry Points
- `workflow.py` → which subpackages it imports
- External loaders (mitmproxy, tmux, etc.) → which file

## Directory Map

| Subdir | Role | LOC | Modules |
|---|---|---|---|
| core/ | Session polling orchestrator | 496 | 3 |
| ... |

## Flow (Main Session)
Numbered 4-6 step pipeline.

## Shared State
Where module-level state lives, who mutates, who reads. Table if multiple state surfaces.

## Root-Level Files
Table: File | LOC | Why at root.

## Subdir DOCS
Links to each subdir DOCS.md.
```

#### Signal-by-design

The format is designed so that scanning the DOCS.md surfaces problems without reading code:

- **LOC per module** → refactor candidate visible (>300 = flag)
- **Called by: []** → dead code flag
- **LOC in Directory Map summed across subdirs** → package-level size comparison
- **State section** → shared mutable surface visible at a glance
- **Gotchas** → landmines documented where they live, not in release notes

#### Rules

- NO function-level documentation (only module-level Purpose/Reads/Writes/Called-by/Calls-out)
- LOC numbers must match actual `wc -l` output of the file
- Called-by must match actual grep results — not stale
- Describe WHAT not HOW
- Include Usage examples for dev/ scripts (how to run from project root)
- Include CLI flags table for scripts with argparse
- When stale: fix before using. Stale DOCS is worse than no DOCS.

### Directories That Do NOT Need DOCS

| Directory | Reason |
|---|---|
| `agents/`, `commands/`, `skills/` | Plugin structure (Claude Code convention) |
| `decisions/` | IS documentation (pipeline decision records) |
| `.claude/`, `.claude-plugin/` | Tool configuration |

### dev/ Suites

dev/ directories follow the same placement rules. Single-script suites are documented in their parent DOCS.md. Multi-script suites get their own DOCS.md.

**Format for dev/ modules (benchmarks, evals, tools):**
- Purpose of the script/suite
- Usage examples (how to run from project root)
- CLI flags if applicable
- Expected output description

### dev/ Investigation Modules

Investigation modules exist to understand and debug a specific problem (e.g., `splade_truncation/`, a corruption bug). They differ from benchmarks/evals because the VALUE is not the scripts — it's the accumulated knowledge about the problem.

**Format for dev/ investigation modules (module-level, NOT per-script):**

```markdown
## module_name/

### Problem

What happens, when, how does it manifest. Symptoms only — not causes.
Production code status quo (fixes, config values) → reference `decisions/` file, NOT here.

### Investigation

#### Code Analysis

Which source files were read, what was found, what's correct/divergent.
Reference file paths and line numbers.

#### External Research

| Source | Result | Relevance |
|--------|--------|-----------|
| Name + URL/Issue# | Found ✅ / Nothing ❌ | Why it matters or doesn't |

#### Hypotheses

| Hypothesis | Status | Evidence |
|------------|--------|----------|
| Description | Active / Excluded / Unverified | What supports or contradicts it |

### Scripts

Per-script: one-liner purpose + usage example. Scripts explain themselves via docstrings.
```

**Key difference to src/ DOCS:** src/ modules document what code DOES. dev/ investigation modules document what we KNOW about a problem — scripts are tools within that investigation.

**When to use:** Any dev/ directory that exists because of a specific bug, performance issue, or behavioral question (not benchmarks or eval suites).

### dev/ vs src/ for Exploratory Rewrites (NON-NEGOTIABLE)

Architectural alternatives — library swaps (httpx vs pydoll), engine rewrites (browser → HTTP), technique replacements, alternative-implementation evaluations — live in `dev/` until empirical evidence has converged on a known-good fix that addresses the actual production problem. `src/` stays untouched during the exploration.

**Pattern:**
- Production module `src/X.py` → UNCHANGED during investigation
- Parallel implementation `dev/<area>/X_<technique>_probe.py` (or sub-suite if multi-file)
- Investigation report `dev/<area>/01_reports/X_<technique>_<YYYYMMDD>.md`
- Smoke test invokes the probe directly, NOT through the production path
- Decision to port to `src/` is a SEPARATE step from the decision to investigate

**Touch src/ only when ALL three hold:**
1. Empirical evidence (smoke / benchmark / live test) confirms the new approach solves the actual production problem
2. The user has green-lit shipping the change
3. The port from dev/ probe to src/ is itself a contained, scoped task — no further architectural exploration mid-port

**Anti-pattern (concrete):** Worker rewrites `src/X.py` directly inside their worktree. Smoke shows the rewrite is architecturally clean but the original production symptom persists (because the root cause was elsewhere). Now we have a fork: merge the rewrite anyway (production gets new untested code without symptom relief) or drop the branch (worktree gets pruned, branch fades, the proof-of-concept evaporates). The dev/ pattern is the structural prevention for this fork.

**Worker prompt rule:** when a Phase A/B prompt asks for an architectural alternative, the prompt MUST specify "build as dev/ probe, do NOT modify src/" unless the user has explicitly green-lit src/ surgery upfront. The default for any phrase like "rewrite X using Y" / "migrate X from A to B" / "swap library Z" is dev/ probe FIRST.

**Does NOT apply to:**
- Scoped bug fixes (the bug location IS the change location — usually src/)
- Refactors with known endpoints (rename, extract function, move module)
- Changes the user has explicitly green-lit as production-targeted up front
- Production-pipeline integration of an already-validated dev/ probe

## External Source Provenance

External sources (papers, vendor docs, GitHub issues/repos, forum threads, web) are cited INLINE in the document that consumes them. There is no central sources registry — provenance lives where it backs a statement, never in a separate file that must be cross-maintained.

- **`decisions/<step>.md`** — cited in the **Quellen** section (the source list) and in **Evidenz** where a source backs a specific measurement/claim. A decision whose state rests entirely on external sources names those sources in Quellen.
- **`decisions/OldThemes/<topic>/`** — cited inline at the point of consultation ("X consulted → finding Y → decision Z"); investigation modules use the **External Research** table (Source | Result | Relevance).
- **`<Project>-reference`** — external sources that warrant full-text RAG indexing (vendor docs, papers) are indexed into the project's reference collection and cited by collection + document name (e.g. `rag-cli-reference: Fusion_Functions_Hybrid_Retrieval`).

Forum sources from MCP plugin searches (reddit-cli-search, LinkedIn) stay inline references only — no web-crawl, no RAG index.
