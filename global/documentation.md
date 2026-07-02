# Documentation Hierarchy

## Language (NON-NEGOTIABLE)

ALL documentation files are written in ENGLISH — always, without exception. Applies to README.md, every DOCS.md, every `decisions/*.md`, every `decisions/OldThemes/**` file, dev/ reports, and code comments. The conversation language is irrelevant: when the user chat is in German, the artifacts are STILL English. No mixed-language files, no German headers, no German prose in any doc.

## Artifact Density

Everything you read or produce as ARTIFACT — code, DOCS.md, skills/, rules/, decisions/, code-comments — is machine-readable and token-dense. Skills especially: a SKILL.md is a dense procedure, never chat prose (the chat-output prose rule applies to user chat only, not to artifacts).

Concrete: tables instead of prose where multiple dimensions need comparison, keywords instead of full sentences, references to `file:line` instead of explanatory paragraphs, no rhetorical filler ("furthermore", "as we can see", "importantly", "it's worth noting"). Where a paragraph IS needed, it is dense — no repetition, no opening sentences that say nothing.

## Sections Are Optional

Every section in DOCS.md, decisions/, and OldThemes is optional. When a point has nothing to say, leave it out — never fill a field just because the template has it. The order and shape are the standard; a section with nothing to say is omitted, not padded.

## No Issue References in Docs

`decisions/*.md`, `decisions/OldThemes/**/*.md`, `**/DOCS.md` NEVER reference issues. The direction is one-way: issues point to docs, docs don't point back — not even an OldThemes entry naming the issue that was part of the flow at the time.

## RAG Collection Layers

Per project: **two** logical collections.

1. **docs** — all internal project documents: `DOCS.md`, `decisions/*.md`, `decisions/OldThemes/**`
2. **reference** — all external sources: vendor docs (e.g. Anthropic API docs), papers, third-party repos

**Canonical naming:**

| Layer | Convention | Example |
|---|---|---|
| docs | `<Project>-docs` | `monitor-cc-docs` |
| reference | `<Project>-reference` | `monitor-cc-reference` |

## Areas

**Naming — one area name across all three surfaces.** An area carries the SAME name in `decisions/<area>.md`, `decisions/OldThemes/<area>/`, and `dev/<area>/`. OldThemes is ALWAYS a subfolder, created as a folder from the start — never a root-level single file. Files inside have free naming (date-based, purpose-based, both OK).

**New-area flow:** create `decisions/<area>.md` (may stay empty at first) → create `decisions/OldThemes/<area>/` once there is process history to put in it (a decision file with no history needs no OldThemes folder — an empty one would not survive git anyway) → once dev work begins, create `dev/<area>/`. All three share the area name.

**Report outputs in dev/.** A dev script that produces a report is numbered (`01_`, `02_`, …), and its report carries the same number — e.g. `01_test.py` → `md/01_testresults.md`. Reports live in a `md/`, `csv/`, or `png/` folder inside `dev/<area>/`, never the console. Scripts that produce no report are not numbered.

## decisions/

Decision records — FINAL STATE ONLY. Each file documents the current production choice (Current State), the evidence backing it (Evidence), and the open questions that remain (Open Questions). What SHOULD change is NOT part of a decision file — a target/recommendation belongs in an issue, not in the documentation. The process that led to the current choice — alternatives evaluated, superseded values, iteration history — lives in `decisions/OldThemes/<area>/`, NOT here. A decision file is the crystallized conclusion; OldThemes is the working memory that produced it.

Rule: any value or comparison that is NOT the current production state belongs in OldThemes, even if it was true last week. decisions/ files do not carry historical alternatives.

**Current State update timing.** Current State in `decisions/<area>.md` reflects current production state. When code changes functionally, Current State MUST be updated in the SAME commit cycle as the code change — never deferred. Code changed but Current State not updated by Recap = drift, blocks session close. Pure refactors with no behavior change update `<package>/DOCS.md` (module shape) but do not touch `decisions/<area>.md` unless a specific path mention becomes stale.

### Structure (MANDATORY)

Every decision file has these sections in order:

#### Current State
What is CURRENTLY in production code. Code paths, config values, behavior. This section describes reality — not what should be.

#### Evidence
The data that BACKS the Current State — measurements from dev/ scripts, external research, benchmarks. For an external source: its actual content/finding plus a short reference to where it came from, not a bare link. For an internal measurement: the key result plus a reference to the dev/ report that produced it. Whenever evidence backing the Current State exists, it MUST be listed here.

**Internal eval results — cite script AND report-MD AND dataset.** When a decision is justified by a measurement run, Evidence carries the key result and cites:
- The dev/ script that produced it (e.g. `dev/retrieval/A_retrieval_eval.py`)
- The report-MD path (e.g. `dev/retrieval/A_retrieval_eval_reports/baseline_2026-04-28.md`)
- The dataset / collection / sample size (e.g. `test_db` on `rag_test`, 250 chunks, 17 queries)

Lift the KEY evidence into decisions/ and OldThemes, never the whole report. The full report stays in `dev/<area>/`, committed and referenced.

#### Open Questions
Unanswered questions about this area that are still open — what is genuinely unresolved. Not a desired target (that is an issue), but an open question.

#### Sources
The COMPLETE list of all external sources consulted — RAG Collection references, papers, URLs. External sources only; internal eval reports belong in Evidence, not here. Division of labor: Evidence carries each source's content with a short reference, Sources is the full list behind those references.

### OldThemes

`decisions/OldThemes/<area>/` is process documentation. It tracks how a decision was reached: alternatives evaluated, superseded values, dead ends, iteration history, open questions per iteration. The corresponding `decisions/<area>.md` only carries the final crystallized choice — OldThemes is the working memory that produced it.

**Purpose split:**

| File | Content | Scope |
|---|---|---|
| `decisions/<area>.md` | Current production state + Evidence that backs it + open questions | Final state only |
| `decisions/OldThemes/<area>/...` | Process: what was tried, what was rejected, what's still open, why the current choice was made | Full history |

**Values in OldThemes ≠ current status quo.** A number measured in iteration 3 is preserved in OldThemes as historical record, not as Current State. Only what is currently in production code lives in `decisions/<area>.md`.

**The decision derives from OldThemes progress.** A decision file crystallizes the conclusion of the process documented in the matching OldThemes folder. While the process there has not converged, the open points live in the decision file's Open Questions. Once it converges, the chosen value lands in Current State and `decisions/<area>.md` cites the matching OldThemes file in Evidence.

**Demotion rule.** When new evidence supersedes an older measurement (new collection, new methodology, new code), the older measurement moves from `decisions/<area>.md` Evidence to `decisions/OldThemes/<area>/`. decisions/ never carries stale evidence; OldThemes preserves it for attribution and traceability.

## DOCS.md

### Placement

DOCS.md lives at the level of the `.py` files it documents — in the directory holding the modules. One DOCS.md per module-bearing directory, documenting the modules at that level.

### DOCS.md Format (Standard)

#### Subdir DOCS.md Structure

```markdown
# src/<package>/

## Role
One paragraph — WHAT this package does in the bigger picture (not HOW), when to touch it, when NOT to touch it.

## Public Interface
What `__init__.py` exports. One line per export. If `__init__.py` is empty: say so and state the actual entry path (e.g. "loaded via `mitmproxy -s`").

## Flow
3-5 lines: data in → processing → data out.

## Modules

### <module>.py (<LOC> LOC)

**Purpose:** one sentence.
**Reads:** data sources (shared state, files, stdin).
**Writes:** outputs (stdout, files, shared state, mutated state).
**Called by:** list of files/packages. Empty list = DEAD CODE, flag explicitly.
**Calls out:** external package dependencies (not stdlib, not `constants`/`utils`).

---

## State
Which module owns the state, who mutates, who reads.

## Gotchas
Module-specific landmines. Direct text. No rule-link references (rules are always invoked).
```

Module-level entries only — no function-level documentation. Each module heading's `<LOC>` matches the file's actual `wc -l`.

### Directories That Do NOT Need DOCS

| Directory | Reason |
|---|---|
| `agents/`, `commands/`, `skills/` | Plugin structure (Claude Code convention) |
| `decisions/` | IS documentation (decision records) |
| `.claude/`, `.claude-plugin/` | Tool configuration |

One DOCS.md format applies everywhere, `src/` and `dev/` alike — no separate dev variant. Sections that do not apply (e.g. Public Interface for a `dev/` script folder with no `__init__.py`) are simply omitted.
