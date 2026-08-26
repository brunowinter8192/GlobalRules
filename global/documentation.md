# Documentation Hierarchy

## Core Rules

### Language (NON-NEGOTIABLE)

**English, always.**
- Every documentation file is written in English, without exception.
- Documentation files include README.md, DOCS.md, process-docs entries, dev/ reports, and code comments.
- For these files the conversation language is irrelevant, because a German chat still produces English artifacts.
- English is also the language of RAG queries, so the docs they index must match it.
- A file that mixes languages is therefore not allowed.

### Artifact Density

**Written for an agent, not a leisurely human reader.**
- An artifact is anything you read or produce outside the chat.
- Artifacts include code, DOCS.md, skills, process-docs, and code comments.
- Missing from that list are rules, because they follow the chat writing form.

**Specific enough to guide, flexible enough to last.**
- Calibrate between two failure modes.
- 1. Over-detail, spelling out every conceivable case with hardcoded edge-case logic.
- Detail at that level ages badly and misleads once the code changes.
- 2. Vagueness that assumes shared context.
- Vagueness tells the reader nothing concrete.

**Structure over prose.**
- Section a document with headers.
- Below the headers, use a table where multiple dimensions compare.
- In table cells and text alike, prefer keywords over full sentences, and file:line references over explanatory paragraphs.

**A few good examples beat an exhaustive list.**
- A few diverse examples show the expected behavior better than a dump of edge cases.
- An example earns its place only when it shows HOW to decide.
- Showing only that a case exists earns no place, so such an example gets cut.

**Unambiguous naming.**
- Make implicit context explicit.
- Explicit context means names the reader cannot mis-resolve.
- A name like `user_id` resolves cleanly where `user` does not.

### Sections Are Optional

**Omit, don't pad.**
- Every section in DOCS.md and a process-docs entry is optional.
- An optional section with nothing to say is left out.
- Leaving it out beats filling a field just because the template has it.
- The template's order and shape are the standard, and an empty section is omitted rather than padded.

### No Issue References

**Docs never point back at issues.**
- Files under process-docs never reference issues.
- Issues point at docs, and the direction stays one-way.
- One-way means even the issue that drove the work stays unnamed in the entry.

### RAG Collection Layers

**Two collections per project.**
- The docs collection holds all internal project documents, meaning DOCS.md and process-docs.
- Next to the docs collection, the reference collection holds all external sources, such as vendor docs and papers.

**Canonical naming:**

| Layer | Convention | Example |
|---|---|---|
| docs | `<Project>-docs` | `monitor-cc-docs` |
| reference | `<Project>-reference` | `monitor-cc-reference` |

## docs

**DOCS.md is the module map.**
- A DOCS.md describes the modules of its directory.
- The description is the only documentation that gets continuously updated.

### Placement

**One DOCS.md per module directory.**
- The file lives at the level of the `.py` modules it documents.
- At that level it sits in the directory holding those modules.

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
- Function-level documentation does not belong in DOCS.md.
- In DOCS.md each module heading's LOC value matches the file's actual `wc -l`.

## process docs

**Always at the project root, always named process-docs.**
- The process-docs folder always sits at the project root.
- At the root it carries the exact name `process-docs/`.
- The name never changes, and the folder never moves into a subdirectory.
- What the folder records is how things were investigated and decided.
- Investigations and decisions cover evaluated alternatives, measurements, dead ends, iteration history, and the reasoning behind chosen values.

**Write-once, not maintained.**
- A process-docs entry is a dated snapshot, written once and never touched again.
- Touching it again is replaced by writing a new entry when code changes or new work lands.
- The new entry leaves the old one untouched.

**No present-tense "current" claims.**
- An entry never asserts present-tense production state, like "X is the production value".
- Production state gets framed as of its date instead, like "as of 2026-06, the sweep showed X".
- A dated value is a historical record rather than a live figure.

**Structured, not a chaotic dump.**
- Entries are dense, organized, English, dated, and thematic.
- The theme decides the folder, so entries organize into `process-docs/<area>/` subfolders.
- Inside a subfolder, file naming is free, so date-based and purpose-based names both work.
- Files are added one per session, so each session writes its own new `.md` into its area.
- A prior session's file is never extended by a follow-up session.
- Follow-ups and one-offs alike stay inside their area folder, because a loose top-level `.md` is not allowed.

**An area is a line of work with its own question.**
- An area runs across sessions and accumulates entries.
- The accumulating entries work on the area's own question and its own measurements.
- The area's name is used identically by issues, `process-docs/<area>/`, and `dev/<area>/`.

**New area vs existing area.**
- Judge the work against the area it refers to.
- The referred area is called the reference area, meaning the existing area whose entries the work builds on.

NEW area — ANY one suffices:

- Is the reference area a foundation for OTHER work too, meaning a shared base rather than the private predecessor of this one follow-up?
- Does the work draw on OTHER areas besides the reference area?
- Does the work depend on NO existing area at all?

EXISTING area (continue it) — ALL three must hold:

- Does the work depend on an existing area's entries?
- Is that area's foundation essentially the foundation of THIS continuation and no other?
- Does the work draw on this ONE area alone?

- Methods and answers within an area may change completely.
- A complete change of approach continues the area, because a pivot is not a new question.
- A question that one entry settles is an entry inside an area rather than its own area.
- An area is also never a maintained list like a backlog, because lists belong in issues.
- Entries in an area folder stay write-once dated snapshots.

**Cross-references point at AREAS, never at single entries.**
- A process-docs entry must not reference another process-docs file by path.
- A specific file is found via RAG over process-docs plus browsing the folder.
- The folder of another area, `process-docs/<area>/`, may be referenced, and that is wanted.
- The referenced area in an issue's `Area:` field is the primary area only.
- Beyond areas, anything may be referenced, such as dev/ reports, src/ symbols, DOCS.md, or external sources.

**Evidence stays inline.**
- State a measurement's key result in the prose itself.
- The key result means the number, the dataset size, and the finding.
- With the finding inline, the entry stands on its own without following any link.
- A link to a dev/ report may back the claim, but it stays optional reading.

## dev reports

**Report outputs in dev/.**
- A dev script that produces a report writes it into `dev/<area>/`.
- Writing to the console instead is not allowed.
- Inside `dev/<area>/` the report goes to `md/`, `csv/`, or `png/`, chosen by output type.
- The report file carries a descriptive name that traces to its producing script.
- Scripts also produce data outputs like raw corpora, and those go into their own type-named folder, for example `jsonl/`.
- Data folders stay separate from report folders and never mix into `md/`.
- Reports and data organize by theme in `dev/<area>/`.
- The area name matches `process-docs/<area>/` where the two align.
