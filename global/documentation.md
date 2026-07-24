# Documentation Hierarchy

## Core Rules

### Language (NON-NEGOTIABLE)

**English, always.**
ALL documentation files are written in ENGLISH, without exception. Applies to README.md, every DOCS.md, every `process-docs/**` file, dev/ reports, and code comments. The conversation language is irrelevant: when the user chat is in German, the artifacts are STILL English. RAG queries are English, so the docs they index are English too. No mixed-language files, no German headers, no German prose in any doc.

### Artifact Density

**Written for an agent, not a leisurely human reader.**
Everything you read or produce as ARTIFACT — code, DOCS.md, skills/, rules/, process-docs/, code-comments — is consumed by an agent under a finite attention budget, not read at leisure by a human. Write the smallest set of high-signal tokens that fully conveys the behavior or information needed. (The chat-output prose rule is the opposite regime — user chat only, never artifacts.)

**Right altitude.**
Calibrate between two failure modes. Too low: brittle over-specification — hardcoded edge-case logic, exhaustive if-else, every conceivable case enumerated; it rots and misleads. Too high: vague hand-waving that assumes shared context and gives no concrete signal. Specific enough to guide the exact behavior, flexible enough to stay robust.

**Minimal ≠ short.**
Give the full information the reader needs to act correctly, then cut everything that is not that. Trim redundancy, restatement, and filler — never trim substance to hit a length.

**Structure over prose.**
Sectioned with headers; a table where multiple dimensions compare; keywords over full sentences; `file:line` and code symbols over explanatory paragraphs. No rhetorical filler ("furthermore", "as we can see", "importantly", "it's worth noting"). Where a paragraph IS needed, it is dense — no repetition, no opening sentence that says nothing.

**Canonical examples, not laundry lists.**
A few diverse, canonical examples that portray the expected behavior beat an exhaustive dump of edge cases — examples are pictures worth a thousand words. An example earns its place only when it shows HOW to decide, not THAT a case exists.

**Unambiguous naming.**
Make implicit context explicit; name so the reader cannot mis-resolve — `user_id`, not `user`.

### Sections Are Optional

**Omit, don't pad.**
Every section in DOCS.md and a process-docs entry is optional. When a point has nothing to say, leave it out — never fill a field just because the template has it. The order and shape are the standard; a section with nothing to say is omitted, not padded.

### No Issue References

**Docs never point back at issues.**
`process-docs/**/*.md` NEVER reference issues. The direction is one-way: issues point to docs, docs don't point back — not even a process-docs entry naming the issue that was part of the flow at the time.

### RAG Collection Layers

**Two logical collections per project.**

1. **docs**
   all internal project documents: `DOCS.md`, `process-docs/**`
2. **reference**
   all external sources: vendor docs (e.g. Anthropic API docs), papers

**Canonical naming:**

| Layer | Convention | Example |
|---|---|---|
| docs | `<Project>-docs` | `monitor-cc-docs` |
| reference | `<Project>-reference` | `monitor-cc-reference` |

## docs

**The module map.**
The `DOCS.md` surface — the ONLY continuously-maintained doc surface.

### Placement

**One DOCS.md per module directory.**
It lives at the level of the `.py` files it documents — in the directory holding the modules, documenting the modules at that level.

### DOCS.md Format

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

**Module-level only.**
No function-level documentation. Each module heading's `<LOC>` matches the file's actual `wc -l`.

## process docs

**Root-anchored, one fixed name.**
`process-docs/` ALWAYS sits at the project root and is ALWAYS named exactly `process-docs/` — never nested in a subdirectory, never renamed. It is the process-documentation surface: how things were investigated and decided — alternatives evaluated, measurements, dead ends, iteration history, the reasoning behind a chosen code value.

**Write-once, not maintained.**
A process-docs entry is a dated snapshot, written once and never touched again. You do NOT go back and edit it when code changes or new work lands — you write a NEW entry.

**No present-tense "current" claims.**
An entry must never assert present-tense current/production state ("X is the production value"). Frame everything as of its date: "as of 2026-06, the sweep showed …". A value measured in an entry is a historical record, not a live figure.

**Structured, not a chaotic dump.**
Entries are dense and organized — English, dated, thematic. Organize by theme in `process-docs/<area>/` subfolders; file naming inside is free (date-based, purpose-based, both OK). **Every area is ONE folder** — how many entries live inside is secondary. No loose top-level `.md`: even a one-off goes into its area folder.

**Area = work strand with its own driving question.**
An area is a line of work that runs across sessions and accumulates entries — its own question under investigation, its own measurements. The area name is the stable pointer used by issues (`Area: <area>`), `process-docs/<area>/`, and `dev/<area>/` — same name on all surfaces.

**New area vs existing area.**
Judge the work against the area it refers to ("the reference area" = the existing area whose entries it builds on).

NEW area — ANY one suffices:

- Is the reference area a foundation for OTHER work too — a shared base, not the private predecessor of this one follow-up?
- Does the work draw on OTHER areas besides the reference area?
- Does the work depend on NO existing area at all?

EXISTING area (continue it) — ALL three must hold:

- Does the work depend on an existing area's entries?
- Is that area's foundation essentially the foundation of THIS continuation and no other?
- Does the work draw on this ONE area alone?

Methods and answers within an area may change completely — a pivot to a new approach CONTINUES the area. A question that one entry settles is an entry inside an area, never its own area.

An area is never a maintained list (backlog, inventory) — that is what issues are for; every entry in an area folder stays a write-once dated snapshot.

**No cross-references to other process-docs.**
A process-docs entry MUST NOT reference another process-docs entry. Thematic grouping is carried by the FOLDER the entry lives in; related process history is found by RAG over `process-docs` + browsing the folder, never by a hardcoded path to another entry. Anything else may be referenced — `dev/` reports, `src/` files/symbols, `DOCS.md`, external sources.

**Evidence stays inline.**
State a measurement's key result in the prose itself — the number, the dataset/sample size, the finding — so the entry stands on its own; a `dev/` report path may back it, but the entry must be readable without following the link.

## dev reports

**Report outputs in dev/.**
A dev script that produces a report writes it to a `md/`, `csv/`, or `png/` folder (by output type) inside `dev/<area>/`, never the console. The report file carries a DESCRIPTIVE name that traces to its producing script. DATA outputs (raw corpora, cached run payloads) go into their own type-named folder (e.g. `jsonl/`) under `dev/<area>/`, kept separate from the report folders and never mixed into `md/`. Organize dev work by theme in `dev/<area>/` — the same area name used in `process-docs/<area>/` where the two align.
