# Environment

## ~/.claude/ is NOT a Git Repo

`~/.claude/` (global rules, shared-rules, plans, plugins cache) is local-only. Nothing in it is tracked, committed, or pushed. Do NOT flag "not committed" for files in `~/.claude/` — that is expected, not a problem.

## No Personal Data in Public Artifacts

**Problem:** GitHub Issues, commits, and tracked files are public. Absolute system paths, usernames, or local config details leak personal information and pollute git history permanently.

**Rule:**
- **Git commits (tracked files):** NEVER commit files containing absolute paths (`/Users/...`, `/home/...`). Use relative paths or variables (`${CLAUDE_PLUGIN_ROOT}`, `${MINERU_PATH}`).
- **GitHub Issues:** NEVER include absolute paths, usernames, or machine-specific details. Use paths relative to project root only.
- **Beads:** Absolute paths are OK (beads are private/local).

**Pre-commit checklist:**
1. Does ANY staged file contain `/Users/` or `/home/`? → Replace with relative path or variable
2. Does it contain usernames or machine names? → Remove
3. Does it reference local config values (ports, IPs, `.env` values)? → Generalize
**Prohibited in tracked files and GitHub Issues:**
- `/Users/<username>/...` or any absolute home path
- Local hostnames, IPs, ports that are machine-specific
- Contents of `.env` files

**Allowed exceptions:**
- Files in `.beads/` (private/local)

**Principle:** Git history is permanent. Once pushed, absolute paths cannot be removed without history rewrite. Prevent at commit time, not after.
