# Code Standards

## Core Rules

**Three defaults hold in every project.**
- debug/ and logs/ folders stay out of version control and must be in .gitignore.
- Emojis stay out of production code, READMEs, DOCS.md, and logs.
- Script console output stays concise.

**Type hints are recommended but optional.**

**Fail fast and let exceptions fly.**
- No try-catch may silently swallow errors that affect business logic.
- A script must fail if it cannot fulfill its purpose.

## Module Layout

**Every module follows INFRASTRUCTURE, then ORCHESTRATOR, then FUNCTIONS.**

| Section | Contents |
|---|---|
| INFRASTRUCTURE | Imports and constants. Functions and logic do not belong here. Module-specific constants live here. Constants shared by two or more modules go in the config module and are imported. |
| ORCHESTRATOR | One function, named `<command>_workflow` for CLI commands and freely named otherwise. It only calls other functions and contains zero functional logic. Conditional workflow execution and parameter routing are allowed. |
| FUNCTIONS | Ordered by call sequence, one responsibility each. Functions may call other functions internally. Every function is reachable from the orchestrator, directly or indirectly. |

**Utility modules are the exception.**
- Constants-only modules, client.py, and helpers may omit the ORCHESTRATOR and FUNCTIONS sections.

## Comment Rules

**The only comment lines are the three section markers.**
- `# INFRASTRUCTURE`, `# ORCHESTRATOR`, `# FUNCTIONS`.

**Every other comment is prohibited.**
- No docstring on a module, class, or function.
- No header above a `def`.
- No trailing explainer on a statement.
- No annotation on an import.

## Import Convention

**Prefer absolute imports.**
- The form is `from src.module.submodule import name`.
- Relative imports stay out of projects that use absolute imports consistently.

## Inter-Module Dependencies

**When module A needs functionality from module B, A imports specific functions from B.**
- Module A's orchestrator calls the imported functions.
- A function used only by another module belongs in that module.

## Naming Conventions

| Element | Convention |
|---|---|
| Domain folders | `src/domain_name/`, snake_case and descriptive |
| Modules | `src/domain/module_name.py`, snake_case |
| Package markers | `src/__init__.py` and `src/domain/__init__.py`, required for imports |
| Documentation | `src/domain/DOCS.md`, one per domain |
