# Documentation Hierarchy

## Language (NON-NEGOTIABLE)

ALL documentation files are written in ENGLISH — always, without exception. Applies to README.md, every DOCS.md, every `decisions/*.md`, every `decisions/OldThemes/**` file, dev/ reports, and code comments. The conversation language is irrelevant: when the user chat is in German, the artifacts are STILL English. No mixed-language files, no German headers, no German prose in any doc.

## Artifact Density

Everything Claude reads or produces as ARTIFACT — code, DOCS.md, rules/, decisions/, code-comments — is machine-readable and token-dense.

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

## decisions/

Pipeline decision records — FINAL STATE ONLY. Each file documents the current production choice (IST), the evidence backing it (Evidenz), and the recommendation for change if any (SOLL). The process that led to the current choice — alternatives evaluated, superseded values, iteration history — lives in `decisions/OldThemes/<topic>/`, NOT here. A decision file is the crystallized conclusion; OldThemes is the working memory that produced it.

Rule: any value or comparison that is NOT the current production state belongs in OldThemes, even if it was true last week. decisions/ files do not carry historical alternatives.

**IST update timing.** IST in `decisions/<step>.md` reflects current production state. When code changes functionally, IST MUST be updated in the SAME commit cycle as the code change — never deferred. Code-changed-without-IST-update at Recap = drift, blocks session close. Pure refactors with no behavior change update `<package>/DOCS.md` (module shape) but do not touch `decisions/<step>.md` unless a specific path mention becomes stale.

**SOLL → IST direction.** IST (production code) changes only after SOLL (the documented target) does — there is no functional production change without a preceding, evidence-backed SOLL. Evidence-backed means dev/, not opinion: architectural alternatives, rewrites, library and technique swaps live in `dev/` as a probe and stay there until empirical evidence (smoke / benchmark / live test) has converged on a known-good fix for the actual production problem. `src/` stays untouched during that exploration — it is changed last, once the evidence justifies the change and the user has green-lit it, never as the place to explore.

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

### Placement

DOCS.md lives at the level of the `.py` files it documents — in the directory holding the modules. One DOCS.md per module-bearing directory, documenting the modules at that level.

### DOCS.md Format (Standard)

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

#### Rules

- NO function-level documentation (only module-level Purpose/Reads/Writes/Called-by/Calls-out)
- LOC numbers must match actual `wc -l` output of the file
- Describe WHAT not HOW
- Include Usage examples for dev/ scripts (how to run from project root)
- Include CLI flags table for scripts with argparse

### Directories That Do NOT Need DOCS

| Directory | Reason |
|---|---|
| `agents/`, `commands/`, `skills/` | Plugin structure (Claude Code convention) |
| `decisions/` | IS documentation (pipeline decision records) |
| `.claude/`, `.claude-plugin/` | Tool configuration |

### dev/ Suites

dev/ scripts are documented at their own level, same as src/. Format additions for dev/ modules (benchmarks, evals, tools):
- Purpose of the script/suite
- Usage examples (how to run from project root)
- CLI flags if applicable
- Expected output description
