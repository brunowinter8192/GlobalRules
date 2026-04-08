# Automation Framework

## Language

**All automation files (CLAUDE.md, SKILL.md, agent instructions, hooks, slash commands) are ALWAYS written in English.**

- Instructions to Claude = English. No exceptions.
- German only in: chat with user, explicit prose examples within docs
- Code, comments, variable names = English
- This applies to ALL config layers: Skills, Slash Commands, Subagents, Hooks

## Stop on Unexpected Problems

**Problem:** Autonomous "fixes" can invalidate entire workflows.

**Rule:**
- On ANY unexpected problem: STOP IMMEDIATELY
- Inform user with clear problem description
- NEVER autonomously "fix" or implement workarounds
- Wait for explicit user decision

**Examples:** NaN values, missing files, unexpected formats, algorithm errors.

**Why:** Better to ask once too often than once too little.

## Editing Automation Files (CRITICAL)

**Applies to:** ALL automation suite files — SKILL.md, agent instructions, hooks, slash commands, CLAUDE.md

**NEVER remove or summarize existing content** — only the user decides what gets deleted. Always EXTEND or ADD, never replace. Edit workflow details live in `iterative-dev/SKILL.md` (IMPROVE phase).

**Prohibited:** Never remove, replace, summarize, or rewrite existing content.

**Principle:** These files are accumulated knowledge. Additions only, deletions by user.

**Edit Workflow:**
1. READ the target file fully
2. SEARCH for overlapping rules/sections
3. If overlap: EXTEND existing section with new content
4. If no overlap: ADD new section, keeping all existing sections untouched
5. If restructuring needed: MOVE text, never delete it

**Automation File Categories:**
1. `~/.claude/rules/*.md` (global behavior — applies to ALL projects, always loaded for Opus AND Workers)
2. `~/.claude/shared-rules/global/*.md` (shared code conventions — symlinked into projects)
3. `~/.claude/shared-rules/mcp/*.md` (shared MCP conventions — symlinked into MCP projects)
4. `<project>/.claude/rules/*.md` (project-specific rules — local files or symlinks to shared-rules)
5. `<project>/CLAUDE.md` (project orientation — structure, components, key files)
6. Plugin files — see **Global Plugins** registry in `~/.claude/rules/plugins.md`
7. `.claude/commands/*.md` (project slash commands)
8. `~/.claude/scripts/` (hooks)

**Rules Loading Mechanism:**
- `~/.claude/rules/` → Claude Code auto-loads for ALL sessions (global). No size limit.
- Project `.claude/rules/` (symlinks) → Claude Code resolves via git worktree link back to main project. Workers in `.claude/worktrees/<name>/` get project rules even though `.claude/rules/` is gitignored.
- SessionStart Hook stdout injection has a ~50KB size limit — outputs larger than this are persisted to disk but NOT injected into context. Not viable for large rule sets.

**Rules Frontmatter Limitations (tested 2026-04-03):**
- `paths` field: Only positive inclusion patterns work (`paths: ["src/**"]`). NO negative patterns (`!.claude/worktrees/**`).
- `globs` field: Does NOT exist. Only `paths` is recognized.
- Rules with unrecognized frontmatter fields or invalid patterns are silently dropped (not loaded at all).
- Worker isolation via frontmatter or hooks is currently NOT possible for global rules.

**Path Clarity:** Always use FULL paths to distinguish global from project:
- Global: `~/.claude/rules/<name>.md`
- Project: `<project>/CLAUDE.md` or `<project>/.claude/rules/<name>.md`
- NEVER write just `CLAUDE.md` without qualifier — ambiguous which one is meant

**Plan Mode Exception:** Automation file edits are ALWAYS allowed in Plan Mode.

**Plan Mode Override:** When the user explicitly asks you to do something in Plan Mode, execute it.

**Plan Mode Safe Operations:** The following are ALWAYS allowed in Plan Mode (non-destructive, no code changes):
- Plugin settings edits (e.g., `settings.local.json` enabledPlugins)
- MCP tool calls (read-only queries to external services)
- Bead operations (`bd list`, `bd comments add`, etc.)
- File reads, searches, exploration

Plan Mode rules are in `~/.claude/rules/plan-mode.md`.

## Improvement Routing

When a process improvement is identified, route it to the correct source:

| Improvement concerns... | Edit in... | Example |
|---|---|---|
| Claude behavior (communication, scoping, questions) | `~/.claude/rules/` | "Don't ask X, do Y instead" |
| Code convention (all projects) | `~/.claude/shared-rules/global/` | "Inline comments prohibited" |
| MCP convention (all MCP projects) | `~/.claude/shared-rules/mcp/` | "server.py pattern" |
| Project-specific constraint | `<project>/.claude/rules/` | "No GPU ops in workers" |
| Plugin logic (Skills, Agents) | Plugin source repo | iterative-dev, reddit, etc. |

**Symlink propagation:** Edits to `~/.claude/shared-rules/` propagate immediately to all projects that symlink those files. No per-project edits needed.

## Where to Fix What

| Process Error | Fix Location |
|---------------|-------------|
| Wrong MCP tool usage / missing subagent dispatch | Domain skill SKILL.md (e.g., `reddit/SKILL.md`) |
| Subagent produces bad output / wrong format | Agent definition (`agents/<name>.md` in plugin source) |
| Subagent missing tool knowledge | Subagent skill (`agent-<name>/SKILL.md`) |
| Phase workflow issue (PLAN/IMPLEMENT/RECAP) | `iterative-dev/SKILL.md` |
| Hook blocks wrong thing / allows wrong thing | `~/.claude/scripts/` |
| General coding behavior | `~/.claude/rules/*.md` or project `CLAUDE.md` |
| Code/MCP convention | `~/.claude/shared-rules/` (symlinked to projects) |
| Slash command workflow broken | `commands/<name>.md` (in plugin source) |

Worker rules (spawning, orchestration, worktrees, merging) are in `~/.claude/rules/workers.md`.
