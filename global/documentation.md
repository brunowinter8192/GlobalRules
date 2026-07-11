# Documentation Hierarchy

## docs and process docs

### General

Rules in this section apply to BOTH surfaces — every `DOCS.md` and every `process-docs/**` file.

#### Language (NON-NEGOTIABLE)

ALL documentation files are written in ENGLISH — always, without exception. Applies to README.md, every DOCS.md, every `process-docs/**` file, dev/ reports, and code comments. The conversation language is irrelevant: when the user chat is in German, the artifacts are STILL English. RAG queries are English, so the docs they index are English too. No mixed-language files, no German headers, no German prose in any doc.

#### Artifact Density

Everything you read or produce as ARTIFACT — code, DOCS.md, skills/, rules/, process-docs/, code-comments — is consumed by an agent under a finite attention budget, not read at leisure by a human. Write the smallest set of high-signal tokens that fully conveys the behavior or information needed. (The chat-output prose rule is the opposite regime — user chat only, never artifacts.)

**Right altitude.** Calibrate between two failure modes. Too low: brittle over-specification — hardcoded edge-case logic, exhaustive if-else, every conceivable case enumerated; it rots and misleads. Too high: vague hand-waving that assumes shared context and gives no concrete signal. Specific enough to guide the exact behavior, flexible enough to stay robust.

**Minimal ≠ short.** Give the full information the reader needs to act correctly, then cut everything that is not that. Trim redundancy, restatement, and filler — never trim substance to hit a length.

**Structure over prose.** Sectioned with headers; a table where multiple dimensions compare; keywords over full sentences; `file:line` and code symbols over explanatory paragraphs. No rhetorical filler ("furthermore", "as we can see", "importantly", "it's worth noting"). Where a paragraph IS needed, it is dense — no repetition, no opening sentence that says nothing.

**Canonical examples, not laundry lists.** A few diverse, canonical examples that portray the expected behavior beat an exhaustive dump of edge cases — examples are pictures worth a thousand words. An example earns its place only when it shows HOW to decide, not THAT a case exists.

**Unambiguous naming.** Make implicit context explicit; name so the reader cannot mis-resolve — `user_id`, not `user`.

#### Sections Are Optional

Every section in DOCS.md and a process-docs entry is optional. When a point has nothing to say, leave it out — never fill a field just because the template has it. The order and shape are the standard; a section with nothing to say is omitted, not padded.

#### No Issue References

`process-docs/**/*.md` and `**/DOCS.md` NEVER reference issues. The direction is one-way: issues point to docs, docs don't point back — not even a process-docs entry naming the issue that was part of the flow at the time.

#### RAG Collection Layers

Per project: **two** logical collections.

1. **docs** — all internal project documents: `DOCS.md`, `process-docs/**`
2. **reference** — all external sources: vendor docs (e.g. Anthropic API docs), papers, third-party repos

**Canonical naming:**

| Layer | Convention | Example |
|---|---|---|
| docs | `<Project>-docs` | `monitor-cc-docs` |
| reference | `<Project>-reference` | `monitor-cc-reference` |

### docs

The `DOCS.md` surface — the module map, and the ONLY continuously-maintained doc surface.

#### Placement

DOCS.md lives at the level of the `.py` files it documents — in the directory holding the modules. One DOCS.md per module-bearing directory, documenting the modules at that level.

#### Freshness — the ONE maintained doc surface

DOCS.md is the only doc that tracks the code's current shape. When code changes functionally (module added/removed, exports, who-calls-whom, LOC), the DOCS.md at that level MUST be updated in the SAME commit cycle as the code change — never deferred. Code changed but DOCS.md not updated by Recap = drift, blocks session close. Adding process history (a new `process-docs/` entry) does not touch DOCS.md.

#### DOCS.md Format (Standard)

##### Subdir DOCS.md Structure

```markdown
# <dir>/

## Role
One paragraph — WHAT this directory does in the bigger picture (not HOW), when to touch it, when NOT to touch it.

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

### process docs

`process-docs/` (project root) is the process-documentation surface: how things were investigated and decided — alternatives evaluated, measurements, dead ends, iteration history, the reasoning behind a chosen code value.

**Write-once, not maintained.** A process-docs entry is a dated snapshot, written once and never touched again. You do NOT go back and edit it when code changes or new work lands — you write a NEW entry. This is the whole point: process docs carry zero ongoing maintenance (unlike a current-state doc that must track every code change). The only continuously-maintained doc surface is DOCS.md.

**No present-tense "current" claims.** Because an entry is never updated, it must never assert present-tense current/production state ("X is the production value") — that goes stale the moment code moves. Frame everything as of its date: "as of 2026-06, the sweep showed …". A value measured in an entry is a historical record, not a live figure.

**Structured, not a chaotic dump.** Entries are dense and organized — English, dated, thematic. Organize by theme in `process-docs/<area>/` subfolders; file naming inside is free (date-based, purpose-based, both OK). **Every area is ONE folder** — how many entries live inside is secondary. No loose top-level `.md`: even a one-off goes into its area folder.

**No cross-links to other docs.** A process-docs entry MUST NOT reference another process-docs entry or any other in-repo documentation file (a DOCS.md, a former decision doc). Those links rot: process-docs folders get reorganized and renamed, and a doc-to-doc path becomes a dead pointer — this happens constantly and is untenable. Thematic grouping is carried by the FOLDER the entry lives in; related process history is found by RAG over `process-docs` + browsing the folder, never by a hardcoded doc path. What an entry MAY reference: `dev/` reports (evidence provenance) and `src/` files/symbols (descriptive context — the code is the ground truth). Those are stable anchors, not doc-to-doc navigation.

**Evidence stays inline.** State a measurement's key result in the prose itself — the number, the dataset/sample size, the finding — so the entry stands on its own; a `dev/` report path may back it, but the entry must be readable without following the link. The resume path is RAG over `process-docs` + reading the code; there is no crystallized-conclusion file to consult.

## dev report structure

**Report outputs in dev/.** A dev script that produces a report writes it to a `md/`, `csv/`, or `png/` folder (by output type) inside `dev/<area>/`, never the console and never into a per-script `NN_reports/` folder. The report file carries a DESCRIPTIVE name that traces to its producing script — dev scripts are NOT numbered. DATA outputs (raw corpora, cached run payloads) are kept separate from reports, never mixed into `md/`. Organize dev work by theme in `dev/<area>/` — the same area name used in `process-docs/<area>/` where the two align.
