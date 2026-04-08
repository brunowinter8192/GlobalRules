# Project Standards

## Version Policy

Always use latest stable version. Before choosing: check official source, present version + rationale, user confirms.

## Logging

1. NO console prints during normal execution (use logging module)
2. Every non-trivial function MUST log
3. Log only actionable events: Status changes, branches, errors, operation results
4. Follow LOGS_MAP.md structure for log file organization (if project has one)

**Setup pattern:**
- Modules: `logger = logging.getLogger(__name__)`
- Entry points (run_*.py, scripts/*.py, workflow.py): `logging.basicConfig(level=logging.INFO, format=...)`
- No duplicate logger setup — entry point configures once, modules inherit
