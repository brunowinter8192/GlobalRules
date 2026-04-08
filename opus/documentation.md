# Documentation Hierarchy

Three documentation layers, each with distinct audience and scope.

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

### Module Documentation Format

```markdown
## module_name.py

**Purpose:** What this module does.
**Input:** What it takes.
**Output:** What it returns.
```

**Rules:**
- NO function-level documentation (only Purpose/Input/Output)
- Describe WHAT not HOW
- Include Usage examples for scripts (how to run from project root)
- Include CLI flags table for scripts with argparse

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
