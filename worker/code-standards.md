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

**Three comment types are allowed, and everything else is prohibited.**

| Type | Rule |
|---|---|
| Section markers | `# INFRASTRUCTURE`, `# ORCHESTRATOR`, `# FUNCTIONS` |
| Function header | One line, WHAT and not HOW, directly above the `def`. Example is `# Load validated customer data from CSV`. |
| Cross-module import | `# From <module>.py: <what it does>` |

**Inline comments are prohibited in particular.**
- A trailing explainer like `df = df.dropna()  # Remove missing values` is the banned pattern.

## Import Convention

**Prefer absolute imports.**
- The form is `from src.module.submodule import name`.
- Document cross-module imports with a comment like `# From src/module.py: what it does`.
- Relative imports stay out of projects that use absolute imports consistently.

## Inter-Module Dependencies

**When module A needs functionality from module B, A imports specific functions from B.**
- Module A's orchestrator calls the imported functions.
- A function used only by another module belongs in that module.

## Error Handling

**try-catch has allowed and prohibited uses.**

| Allowed | Prohibited |
|---|---|
| Retry with exponential backoff | Silently swallowing errors |
| Graceful degradation with explicit logging | Generic `except Exception: pass` |
| Resource cleanup for files and connections | Hiding failures that affect business logic |
| Converting exceptions to domain errors | |

## Naming Conventions

| Element | Convention |
|---|---|
| Domain folders | `src/domain_name/`, snake_case and descriptive |
| Modules | `src/domain/module_name.py`, snake_case |
| Package markers | `src/__init__.py` and `src/domain/__init__.py`, required for imports |
| Documentation | `src/domain/DOCS.md`, one per domain |
