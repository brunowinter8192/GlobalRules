# MCP Server Pattern

## server.py Structure

```python
# INFRASTRUCTURE
import asyncio
from fastmcp import FastMCP
from mcp.types import TextContent

from src.domain.module import module_workflow

mcp = FastMCP("PluginName")

# TOOL REGISTRATION

@mcp.tool
def tool_name(param: str) -> list[TextContent]:
    """One-line tool description for the LLM."""
    return asyncio.run(module_workflow(param))

if __name__ == "__main__":
    mcp.run()
```

## Rules

1. **Section markers:** `# INFRASTRUCTURE` then `# TOOL REGISTRATION` — no ORCHESTRATOR/FUNCTIONS (server.py is glue, not logic)
2. **Tool functions are thin wrappers** — one line: `return asyncio.run(workflow(...))`. Zero logic in server.py. Validation, routing, formatting all live in `src/`.
3. **Pre-routing pattern** — when URLs must be intercepted before processing (plugin routing, blocklists), call the check before the workflow:
   ```python
   if blocked := check_plugin_routed(url):
       return blocked
   return asyncio.run(workflow(url))
   ```
4. **Return type** — always `list[TextContent]`. Workflows in `src/` construct and return `TextContent` objects.
5. **Async bridge** — `asyncio.run()` wraps async workflows. Use `nest_asyncio.apply()` at module level if the event loop is already running.
6. **No state in server.py** — no globals, no caches, no session management. State lives in `src/` modules.
7. **Tool docstrings** — one sentence, describes what the LLM gets. This is the description shown to Claude — keep it action-oriented and concise.
8. **Parameter types** — use `Literal[...]` for enum params, `str | None` for optional. FastMCP generates the JSON schema from type hints.
