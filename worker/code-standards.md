# Code Standards

- NO comments inside function bodies (only function header comments + section markers)
- NO test files in root (ONLY in debug/ folders - root or per-module)
- NO debug/ or logs/ folders in version control (MUST be in .gitignore)
- NO emojis in production code, READMEs, DOCS.md, logs
- ALWAYS keep script console output concise

**Type hints:** RECOMMENDED but optional

**Fail-Fast:** Let exceptions fly. No try-catch that silently swallows errors affecting business logic. Script must fail if it cannot fulfill its purpose.

# Error Handling

## When to use try-catch
**ALLOWED:**
- Retry logic with exponential backoff
- Graceful degradation with explicit logging
- Resource cleanup (files, connections)
- Converting exceptions to domain errors

**PROHIBITED:**
- Silently swallowing errors
- Generic `except Exception: pass`
- Hiding failures that affect business logic

# Naming Conventions

**Domain folders:** src/domain_name/ (snake_case, descriptive)
**Modules:** src/domain/module_name.py (snake_case)
**Package markers:** src/__init__.py and src/domain/__init__.py (required for imports)
**Documentation:** src/domain/DOCS.md (one per domain)
