# Documentation Hierarchy

## Language (NON-NEGOTIABLE)

ALL documentation files are written in ENGLISH — always, without exception. Applies to README.md, every DOCS.md, every `process-docs/**` file, dev/ reports, and code comments. The conversation language is irrelevant: when the user chat is in German, the artifacts are STILL English. RAG queries are English, so the docs they index are English too. No mixed-language files, no German headers, no German prose in any doc.

## Artifact Density

Everything you read or produce as ARTIFACT — code, DOCS.md, skills/, rules/, process-docs/, code-comments — is machine-readable and token-dense. Skills especially: a SKILL.md is a dense procedure, never chat prose (the chat-output prose rule applies to user chat only, not to artifacts).

Concrete: tables instead of prose where multiple dimensions need comparison, keywords instead of full sentences, references to `file:line` instead of explanatory paragraphs, no rhetorical filler ("furthermore", "as we can see", "importantly", "it's worth noting"). Where a paragraph IS needed, it is dense — no repetition, no opening sentences that say nothing.

## Sections Are Optional

Every section in DOCS.md and a process-docs entry is optional. When a point has nothing to say, leave it out — never fill a field just because the template has it. The order and shape are the standard; a section with nothing to say is omitted, not padded.

## No Issue References in Docs

`process-docs/**/*.md` and `**/DOCS.md` NEVER reference issues. The direction is one-way: issues point to docs, docs don't point back — not even a process-docs entry naming the issue that was part of the flow at the time.

## RAG Collection Layers

Per project: **two** logical collections.

1. **docs** — all internal project documents: `DOCS.md`, `process-docs/**`
2. **reference** — all external sources: vendor docs (e.g. Anthropic API docs), papers, third-party repos

**Canonical naming:**

| Layer | Convention | Example |
|---|---|---|
| docs | `<Project>-docs` | `monitor-cc-docs` |
| reference | `<Project>-reference` | `monitor-cc-reference` |

## Current State Lives in Code

There is **no documentation mirror of current production state.** The source code IS the current state — authoritative, never duplicated into a doc that can drift. "What IS live now" is answered by reading the code, not a doc. What SHOULD change — targets, recommendations, open questions — lives in **issues**, never in docs. The only doc that tracks the code's current shape is DOCS.md (module map), and it is maintained in lockstep with the code (see DOCS.md § Freshness).

## process-docs/

`process-docs/` (project root) is the process-documentation surface: how things were investigated and decided — alternatives evaluated, measurements, dead ends, iteration history, the reasoning behind a chosen code value.

**Write-once, not maintained.** A process-docs entry is a dated snapshot, written once and never touched again. You do NOT go back and edit it when code changes or new work lands — you write a NEW entry. This is the whole point: process docs carry zero ongoing maintenance (unlike a current-state doc that must track every code change). The only continuously-maintained doc surface is DOCS.md.

**No present-tense "current" claims.** Because an entry is never updated, it must never assert present-tense current/production state ("X is the production value") — that goes stale the moment code moves. Frame everything as of its date: "as of 2026-06, the sweep showed …". A value measured in an entry is a historical record, not a live figure.

**Structured, not a chaotic dump.** Entries are dense and organized — English, dated, thematic. Organize by theme in `process-docs/<area>/` subfolders; file naming inside is free (date-based, purpose-based, both OK). **Every area is ONE folder** — how many entries live inside is secondary. No loose top-level `.md`: even a one-off goes into its area folder.

**No cross-links to other docs.** A process-docs entry MUST NOT reference another process-docs entry or any other in-repo documentation file (a DOCS.md, a former decision doc). Those links rot: process-docs folders get reorganized and renamed, and a doc-to-doc path becomes a dead pointer — this happens constantly and is untenable. Thematic grouping is carried by the FOLDER the entry lives in; related process history is found by RAG over `process-docs` + browsing the folder, never by a hardcoded doc path. What an entry MAY reference: `dev/` reports (evidence provenance) and `src/` files/symbols (descriptive context — the code is the ground truth). Those are stable anchors, not doc-to-doc navigation.

**Evidence stays inline.** State a measurement's key result in the prose itself — the number, the dataset/sample size, the finding — so the entry stands on its own; a `dev/` report path may back it, but the entry must be readable without following the link. The resume path is RAG over `process-docs` + reading the code; there is no crystallized-conclusion file to consult.

## dev/ Layout

**Report outputs in dev/.** A dev script that produces a report is numbered (`01_`, `02_`, …), and its report carries the same number — e.g. `01_test.py` → `md/01_testresults.md`. Reports live in a `md/`, `csv/`, or `png/` folder inside `dev/<area>/`, never the console. Scripts that produce no report are not numbered. Organize dev work by theme in `dev/<area>/` — the same area name used in `process-docs/<area>/` where the two align.

## DOCS.md

### Placement

DOCS.md lives at the level of the `.py` files it documents — in the directory holding the modules. One DOCS.md per module-bearing directory, documenting the modules at that level.

### Freshness — the ONE maintained doc surface

DOCS.md is the only doc that tracks the code's current shape. When code changes functionally (module added/removed, exports, who-calls-whom, LOC), the DOCS.md at that level MUST be updated in the SAME commit cycle as the code change — never deferred. Code changed but DOCS.md not updated by Recap = drift, blocks session close. Adding process history (a new `process-docs/` entry) does not touch DOCS.md.

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
| `process-docs/` | IS documentation (process records) |
| `.claude/`, `.claude-plugin/` | Tool configuration |

One DOCS.md format applies everywhere, `src/` and `dev/` alike — no separate dev variant. Sections that do not apply (e.g. Public Interface for a `dev/` script folder with no `__init__.py`) are simply omitted.
