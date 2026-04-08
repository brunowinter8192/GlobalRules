# Plugins

## Global (User Scope — always available)

| Plugin | Content |
|--------|---------|
| `iterative-dev` | iterative-dev Skill, plugin-dev Skill, eval-agent Skill, doc-review Skill, agent-code-investigate Skill, code-investigate-specialist Agent, git-committer Agent, eval-spawn Command, docs-review Command |

## Local (Project Scope — installed per project as needed)

| Plugin | Content |
|--------|---------|
| `rag` | agent-rag-search Skill, agent-md-cleanup Skill, agent-web-md-cleanup Skill, rag-search Agent, md-cleanup-master Agent, web-md-cleanup Agent, pdf-convert Command, web-md-index Command, MCP Server |
| `github-research` | agent-github-search Skill, github-search Agent, MCP Server |
| `reddit` | agent-reddit-search Skill, reddit-commenting Skill, reddit-search Agent, MCP Server |
| `searxng` | agent-web-research Skill, web-research Agent, crawl-site Command, MCP Server |
| `arxiv` | agent-arxiv-search Skill, arxiv-search Agent, MCP Server |

## Agent vs Skill Content

**System Prompt** (`agents/<name>.md`) = autonomous behavior — HOW the agent operates:
- Autonomous operation rules (no questions, start with tool call)
- Report format (output structure expected by caller)
- When to stop
- Content filters

**Skill** (`skills/agent-<name>/SKILL.md`) = tool knowledge — WHAT the agent needs to know:
- Tool reference tables + parameter documentation
- Usage strategies + search workflows
- Server prerequisites (GPU start, health checks)
- Known limitations + guardrails
- Domain-specific rules (honesty for RAG, navigation for GitHub)

**Never put workflow phases or tool strategies in the system prompt.**

**Plugin Source Discovery:**
```bash
find ~/.claude/plugins/cache/brunowinter-plugins/ -name "plugin.json"
```

**When improving Automation Files during RECAP:**
1. Identify which plugin the file belongs to
2. Locate plugin source via discovery command above
3. Edit at source — not in project local copies

**NEVER edit files in `~/.claude/plugins/cache/` directly.** The cache is a copy, not the source. Edits in cache get overwritten by the next `plugin-sync.sh` run. Always edit in the plugin source repo (e.g., `~/Documents/ai/Meta/blank` for iterative-dev, `~/Documents/ai/Meta/ClaudeCode/MCP/Reddit` for reddit), then commit, push, and run `plugin-sync.sh`.

## Plugin Cache Management

**Sync local plugin repo → cache (preferred over `/plugin install`):**
```bash
~/.claude/plugins/cache/brunowinter-plugins/iterative-dev/1.0.0/plugin-sync.sh <name> <repo-path>
```

**Examples:**
```bash
plugin-sync.sh rag ~/Documents/ai/Meta/ClaudeCode/MCP/RAG
plugin-sync.sh reddit ~/Documents/ai/Meta/ClaudeCode/MCP/Reddit
plugin-sync.sh iterative-dev ~/Documents/ai/Meta/blank
```

**After sync:** Kill running MCP server processes (`ps aux | grep <plugin>.*server.py | xargs kill`), then `/mcp` to reconnect. Full session restart only if `/mcp` doesn't pick up changes.

**Edit-Deploy Cycle (CRITICAL — MCP Plugins):**
When editing MCP plugin source files (server.py, src/*.py, requirements.txt):
1. Edit in SOURCE REPO (not cache)
2. `plugin-sync.sh <name> <repo-path>` IMMEDIATELY after edit
3. Kill old server: `ps aux | grep "<name>.*server.py" | grep -v grep | awk '{print $2}' | xargs kill`
4. `/mcp` to reconnect (starts fresh server with new code)
5. THEN test

**NEVER** test MCP changes without sync+kill first — the running server uses cached code in memory.

**Enable/disable plugin per project** (in `<project>/.claude/settings.local.json`):
```json
{
  "enabledPlugins": {
    "github-research@brunowinter-plugins": true
  }
}
```
Set `true` to enable, remove entry to disable. Plugin must be installed globally (`scope: "user"`).

**Plugin Activation Rule:**
- When user says "activate plugin X" → add `enabledPlugins` entry to project's `settings.local.json` IMMEDIATELY
- No questions, no confirmation — just do it
- Format: `"<plugin-name>@brunowinter-plugins": true`
