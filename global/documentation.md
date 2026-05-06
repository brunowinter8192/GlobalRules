# Documentation Hierarchy

Three documentation layers, each with distinct audience and scope.

## Artifact Density

User-Chat is prose (see `global/chat-output.md`). Everything Claude reads or produces as ARTIFACT — code, DOCS.md, CLAUDE.md, rules/, decisions/, code-comments — is machine-readable and token-dense.

Concrete: tables instead of prose where multiple dimensions need comparison, keywords instead of full sentences, references to `file:line` instead of explanatory paragraphs, no rhetorical filler ("furthermore", "as we can see", "importantly", "it's worth noting"). Where a paragraph IS needed, it is dense — no repetition, no opening sentences that say nothing.

The user reads code and docs through Claude. Token-dense input leaves more context budget for actual work. Every line of prose in a DOCS.md or rule is context cost without information gain over the structured alternative.

Applies to every project, not only Monitor_CC.

## CLAUDE.md (Root)

**Audience:** AI (Claude Code sessions).
**Purpose:** Maps root level for AI navigation. Contains project overview, pipeline components, key files, startup instructions.
**Scope:** Root-level structure. References `data/` and `decisions/` with their purpose. Links to DOCS.md for module details.

**Not a substitute for DOCS.md.** CLAUDE.md provides orientation; DOCS.md provides module documentation.

## README.md (Root)

**Audience:** External user (human). Someone who finds the repo on GitHub.
**Purpose:** Understand what this is, decide if they need it, get it running.
**Scope:** User-facing only. No internal architecture, no directory trees, no stack decisions.

**Section Order:**

### 1. Header + One-Liner
Plugin name + one sentence: what problem it solves.

### 2. Features
Bullet list of capabilities. What the user GETS, not how it works internally.

### 3. Quick Start
3-5 copy-paste lines to install and use. Plugin install → restart → first command.

### 4. Prerequisites
What must exist before setup (Docker, Python, hardware). Note what auto-starts.

### 5. Setup
Step-by-step manual installation. Copy-paste commands. Reference .env.example for config.

### 6. Usage

#### MCP Tools
Table: Tool | What it does | Example prompt. User perspective, not API signatures.

#### Skills & Commands
Which slash commands the plugin provides, what each does, when to use it.

#### Agents
Which subagents the plugin includes, what they handle, how they get dispatched.

### 7. Workflows
Brief explanation of main workflows (e.g., PDF→RAG pipeline). Enough to understand the flow, not implementation details. Detailed docs → DOCS.md.

### 8. Troubleshooting
`<details>` collapsible blocks for common problems. Problem → Solution format.

### 9. License

**Rules:**
- NO directory trees (→ CLAUDE.md)
- NO stack decisions or architecture rationale (→ decisions/)
- NO internal module docs (→ DOCS.md)
- NO function signatures or code internals
- Environment variables: maintain .env.example in repo, README references it
- Keep it scannable: someone should understand the plugin in 2 minutes
- README stops where DOCS begins. No redundancy.

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
| `data/` | Data storage, purpose documented in CLAUDE.md |
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

## sources/sources.md

Tracks all external sources referenced or indexed during research.

### Format

```
| Source | Domain | Type | Decision Steps | Status |
```

- **Type:** `Repo` | `Web` | `Paper` | `Thesis` | `Forum`
- **Status:** `Referenced` | `Verified` | `Indexed (RAG: <collection>)`

### Status Rules

| Status | Bedeutung |
|--------|-----------|
| Referenced | Quelle erwähnt/genutzt, kein RAG-Index |
| Verified | URL und Inhalt manuell bestätigt |
| Indexed (RAG: x) | In RAG-Collection `x` indexiert |

### Plugin-Suchen (Forum)

Forum-Quellen aus MCP-Plugin-Suchen (reddit-search-Agent, LinkedIn-Suche) bleiben immer **Referenced** — kein Web-Crawl, kein RAG-Index. Status wird nicht auf Indexed gesetzt.
