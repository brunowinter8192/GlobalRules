# Environment

## ~/.claude/ is NOT a Git Repo

`~/.claude/` (global rules, shared-rules, plans, plugins cache) is local-only. Nothing in it is tracked, committed, or pushed. Do NOT flag "not committed" for files in `~/.claude/` — that is expected, not a problem.

## No Personal Data in Public Artifacts

Git history is permanent. Once pushed, absolute paths and machine-specific details cannot be removed without history rewrite — prevent at commit time.

**Rule:** Tracked files and GitHub Issues must NEVER contain:
- Absolute home paths (`/Users/...`, `/home/...`) → use relative paths or variables (`${CLAUDE_PLUGIN_ROOT}`, `${MINERU_PATH}`)
- Usernames, hostnames, machine-specific IPs/ports
- Contents of `.env` files

**Exception:** Files under `.beads/` are private/local — absolute paths OK there.

## User CLI Wrappers

**Default location for user-scope CLI wrapper scripts: `~/.local/bin/`.**

`~/.local/bin/` is the XDG Base Directory standard for user-installed executables and
is typically already in PATH — including in Claude's Bash tool environment (which does
NOT source `~/.zshrc`). Any other location (like `~/bin/`) requires a PATH export in
`.zshrc` that won't reach Claude's non-interactive shell.

**Rule:**
- New user CLI wrappers → write directly to `~/.local/bin/<name>` + `chmod +x`. Done.
- Do NOT create `~/bin/` unless the user explicitly asks for it.
- Do NOT add PATH exports to `~/.zshrc` for Claude-facing tooling — the export won't
  propagate to the Bash tool, so you'd still need absolute paths or symlinks anyway.

**Verification before picking a wrapper location:**
```
echo "$PATH" | tr ':' '\n' | grep -E '(local/bin|/bin$)'
```
Run this in Claude's Bash tool (not your interactive shell) to see what's actually in
PATH. Pick a location from that list. If nothing user-writeable is listed, `~/.local/bin`
is the default — create the dir if missing.

Concrete failure (2026-04-15): Placed 4 research plugin wrappers in `~/bin/` with
`export PATH="$HOME/bin:$PATH"` appended to `~/.zshrc`. User's interactive shell picked
it up; Claude's Bash tool did not (no `.zshrc` sourcing in non-interactive mode). Second
migration to `~/.local/bin/` required — including `mv` 4 files, revert `.zshrc` export,
`rmdir ~/bin`. Option C (`~/.local/bin` from the start) would have been one step.
