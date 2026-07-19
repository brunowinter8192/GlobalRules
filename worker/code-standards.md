# Code Standards

## Core Rules

- NO debug/ or logs/ folders in version control (MUST be in .gitignore)
- NO emojis in production code, READMEs, DOCS.md, logs
- ALWAYS keep script console output concise

**Type hints are RECOMMENDED but optional.**

**Fail-Fast — let exceptions fly.**
No try-catch that silently swallows errors affecting business logic. Script must fail if it cannot fulfill its purpose.

## Module Layout

Every module follows INFRASTRUCTURE → ORCHESTRATOR → FUNCTIONS.

| Section | Contents |
|---|---|
| INFRASTRUCTURE | Imports and constants — NO functions, NO logic. Module-specific constants live here; constants shared by 2+ modules go in the config module (`config.py`/`constants.py`) and are imported. |
| ORCHESTRATOR | ONE function (`<command>_workflow` for CLI commands, freely named otherwise). Calls only — ZERO functional logic (no calculations, transformations, business rules). Meta-logic allowed: conditional workflow execution, parameter routing. |
| FUNCTIONS | Ordered by call sequence, one responsibility each, may call other functions internally; every function reachable from the orchestrator (directly or indirectly). |

**Exception — utility modules.**
Constants-only, client.py, helpers may omit ORCHESTRATOR and/or FUNCTIONS sections.

## Comment Rules

Three comment types are allowed; everything else — especially inline comments — is prohibited.

| Type | Rule |
|---|---|
| Section markers | `# INFRASTRUCTURE`, `# ORCHESTRATOR`, `# FUNCTIONS` |
| Function header | One line, WHAT not HOW, directly above the `def`: `# Load validated customer data from CSV` |
| Cross-module import | `# From <module>.py: <what it does>` |

**PROHIBITED — inline comments.**
No trailing explainers like `df = df.dropna()  # Remove missing values`.

## Import Convention

- Prefer absolute imports (`from src.module.submodule import name`)
- Document cross-module imports with comment: `# From src/module.py: what it does`
- No relative imports in projects that use absolute import style consistently

## Inter-Module Dependencies

When module A needs functionality from module B:
- Module A imports specific functions from module B
- Module A's orchestrator calls imported functions
- If a function is only used by another module, it belongs in THAT module

## Error Handling

try-catch, when to use it and when not:

| Allowed | Prohibited |
|---|---|
| Retry with exponential backoff | Silently swallowing errors |
| Graceful degradation with explicit logging | Generic `except Exception: pass` |
| Resource cleanup (files, connections) | Hiding failures that affect business logic |
| Converting exceptions to domain errors | — |

## Naming Conventions

| Element | Convention |
|---|---|
| Domain folders | `src/domain_name/` — snake_case, descriptive |
| Modules | `src/domain/module_name.py` — snake_case |
| Package markers | `src/__init__.py` and `src/domain/__init__.py` — required for imports |
| Documentation | `src/domain/DOCS.md` — one per domain |
