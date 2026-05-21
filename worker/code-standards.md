# Code Standards

- NO test files in root (ONLY in dev/ folders - root or per-module)
- NO debug/ or logs/ folders in version control (MUST be in .gitignore)
- NO emojis in production code, READMEs, DOCS.md, logs
- ALWAYS keep script console output concise

**Type hints:** RECOMMENDED but optional

**Fail-Fast:** Let exceptions fly. No try-catch that silently swallows errors affecting business logic. Script must fail if it cannot fulfill its purpose.

## Error Handling

### When to use try-catch
**ALLOWED:**
- Retry logic with exponential backoff
- Graceful degradation with explicit logging
- Resource cleanup (files, connections)
- Converting exceptions to domain errors

**PROHIBITED:**
- Silently swallowing errors
- Generic `except Exception: pass`
- Hiding failures that affect business logic

## Immutability

### Functions Must Not Mutate Their Arguments

A function that produces a modified collection returns a NEW value — mutating caller-passed dicts, lists, or objects is prohibited.

```python
# WRONG — modifies caller's dict silently
def _extract_fields(entry):
    entry['parsed'] = parse(entry['raw'])

# RIGHT — explicit return
def _extract_fields(entry):
    return {**entry, 'parsed': parse(entry['raw'])}
```

Building local collections with `result = []; result.append(...); return result` is fine — `result` is local, not an argument.

### Module-Level Mutable State

Permitted when ALL of these hold:

1. State is owned by one module — only that module mutates it.
2. State is documented in the module's DOCS.md State section.
3. External readers access via accessor functions, never via direct mutation of an imported module's globals.

## Naming Conventions

**Domain folders:** src/domain_name/ (snake_case, descriptive)
**Modules:** src/domain/module_name.py (snake_case)
**Package markers:** src/__init__.py and src/domain/__init__.py (required for imports)
**Documentation:** src/domain/DOCS.md (one per domain)
