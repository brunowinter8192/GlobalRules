# MCP Testing

## Workers Cannot Test Via MCP

Workers in worktrees have NO running MCP server. They cannot make MCP tool calls.

**Worker testing options:**
1. **Import from src/** — call workflow functions or sync wrappers directly
2. **Syntax check** — `python -c "import ast; ast.parse(open('file').read())"`
3. **Import check** — `python -c "from src.domain.module import workflow"`

**Live MCP testing** (server restart, tool calls) is the parent session's responsibility after merge.

## Dev Script Testing Pattern

Dev scripts in `dev/` test production logic by importing from `src/`:

```python
from src.domain.module import sync_wrapper_function

results = sync_wrapper_function(query)
```

- Import the sync wrapper, not the async workflow
- Never import from `server.py`
- Never instantiate FastMCP in dev scripts

## Test Isolation

- Dev scripts manage their own browser/client instances when needed
- Production singletons (shared browser, rate limiters) may interfere — dev scripts that need isolation should instantiate fresh
- Output goes to `NN_reports/` directories, never to console (see dev-convention.md)
