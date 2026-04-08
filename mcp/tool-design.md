# MCP Tool Design

## Workflow Pattern

Every MCP tool maps to exactly one `*_workflow()` function in `src/`.

```
server.py → @mcp.tool thin wrapper
  └── src/domain/module.py → module_workflow() ORCHESTRATOR
        ├── _helper_1()
        ├── _helper_2()
        └── _format_results() → list[TextContent]
```

## Rules

1. **One workflow per tool** — `search_web` → `search_web_workflow()`, `scrape_url` → `scrape_url_workflow()`. Naming: `<tool_name>_workflow`.
2. **Workflow is the ORCHESTRATOR** — follows the standard INFRASTRUCTURE/ORCHESTRATOR/FUNCTIONS structure. The workflow function is the single orchestrator.
3. **TextContent construction in src/** — the workflow builds and returns `list[TextContent]`. server.py never constructs TextContent.
4. **Private helpers** — prefix with `_`. Ordered by call sequence under `# FUNCTIONS`.
5. **Error handling at workflow level** — the workflow catches domain errors and returns error TextContent. Helpers raise exceptions, workflow catches and formats.
6. **Sync wrapper for dev scripts** — when dev scripts need to call the same logic, provide a sync function alongside the async workflow:
   ```python
   # Synchronous wrapper for dev scripts
   def fetch_results(query: str) -> list[dict]:
       return asyncio.run(_async_impl(query))
   ```
   Dev scripts import the sync wrapper, not the workflow (which returns TextContent).

## Parameter Design

- **Required params first**, optional with defaults after
- **Sane defaults** — tool must work with just required params
- **No internal params exposed** — rate limits, timeouts, retry counts stay in `src/`, not in tool signature
- **Language/locale** — default to `"en"`, accept as optional param
