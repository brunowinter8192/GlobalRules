# CLAUDE.md Convention

## Structure (MANDATORY)

Every project CLAUDE.md follows this structure in order:

```
# {Project Name}
{One-line description}

## Sources
See [sources/sources.md](sources/sources.md).

## Pipeline Components
### {Stage 1 Name}
| Component | Implementation | Config |
### {Stage 2 Name}
| Component | Implementation | Config |
...
### Key Files
| File | Component |

## Project Structure
{tree with DOCS.md links}
```

## Rules

1. **Tables only** — Pipeline info lives in `Component | Implementation | Config` tables. No prose bullets, no flow diagrams, no redundant explanations.
2. **No detail duplication** — Implementation details live in `decisions/` files. CLAUDE.md shows WHAT exists and WHERE, not HOW it works. Config column has concise values (port numbers, model names, limits), not explanations.
3. **Bidirectional mapping** — Every pipeline component in CLAUDE.md has a corresponding `decisions/` file. Every `decisions/` file corresponds to a pipeline component in CLAUDE.md. Every `decisions/` file has a corresponding `dev/` suite directory.
4. **No discoverable sections** — Automation (Skills, Agents, Commands) is discoverable from `.claude/agents/`, `.claude/skills/`, `.claude/commands/`. Decision↔Dev mapping lives in `dev/DOCS.md`. Neither belongs in CLAUDE.md.
5. **Key Files table** — Maps source files to pipeline components. Only files that a new session needs to navigate the codebase.
6. **Project Structure tree** — Shows directory layout with `→ [DOCS.md](path)` links. `decisions/` files listed here (only place they appear in CLAUDE.md).
7. **Reference pattern** — RAG/CLAUDE.md is the canonical example.
