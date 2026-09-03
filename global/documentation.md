# Documentation Hierarchy

## Core Rules

### Language (NON-NEGOTIABLE)

**English, always.**
- Every documentation file is written in English, without exception.
   - Documentation files include DOCS.md, process-docs entries, dev/ reports, and code comments.

### Artifact Density

**Written for an agent, not a leisurely human reader.**
- An artifact is anything you read or produce outside the chat.
   - Artifacts include code, DOCS.md, skills, process-docs, and code comments.

**Specific enough to guide, flexible enough to last.**
- Write the concrete points a reader needs to act, so nothing rests on shared context.
- Keep the detail at the level that survives the next code change.

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

### No Issue References

**Docs never point back at issues.**
- Files under process-docs never reference issues.
- Issues point at docs, and the direction stays one-way.

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
   - That description is the only documentation that gets continuously updated.

### Placement

**One DOCS.md per module directory.**
- The file lives in the directory holding the `.py` modules it documents.

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
- What the folder records is how things were investigated and decided.

**Write-once, not maintained.**
- A process-docs entry is a dated snapshot, written once and never touched again.
   - New work gets a NEW entry instead of touching the old one.

**No present-tense "current" claims.**
- An entry never asserts present-tense production state, like "X is the production value".
- Production state gets framed as of its date instead, like "as of 2026-06, the sweep showed X".

**Structured, not a chaotic dump.**
- The theme decides the folder, so entries organize into `process-docs/<area>/` subfolders.
   - Inside a subfolder, file naming is free, so date-based and purpose-based names both work.

**An area is a line of work.**
- An area runs across sessions and accumulates entries.
- The area's name is used identically by issues, `process-docs/<area>/`, and `dev/<area>/`.

**Methods and answers within an area may change completely.**
- A complete change of approach continues the area, because a pivot is not a new question.

**Cross-references point at AREAS, never at single entries.**
- A process-docs entry must not reference another process-docs file by path.
- The folder of another area, `process-docs/<area>/`, may be referenced, and that is wanted.

**Evidence stays inline.**
- State a measurement's key result in the prose itself.
   - The key result means the number, the dataset size, and the finding.
- A link to a dev/ report may back the claim, but it stays optional reading.
