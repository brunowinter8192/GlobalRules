# MCP src/ Structure

## Domain Organization

```
src/
├── __init__.py
├── routing.py              → Cross-domain utilities (plugin routing, shared helpers)
├── domain_a/               → One pipeline domain
│   ├── __init__.py
│   ├── module.py           → Contains *_workflow() called by server.py
│   ├── helper.py           → Domain-internal helpers (optional)
│   └── DOCS.md
├── domain_b/
│   ├── __init__.py
│   ├── module.py
│   └── DOCS.md
```

## Rules

1. **One directory per pipeline domain** — search, scraper, crawler, indexer, etc. Each maps to one or more MCP tools.
2. **Flat within domain** — modules directly in the domain dir, no nested subdirectories unless the domain has genuinely independent sub-systems (e.g., `search/engines/` with multiple engine implementations).
3. **`__init__.py` required** — in `src/` and every domain directory. Needed for absolute imports (`from src.domain.module import ...`).
4. **Cross-domain imports via src/** — `from src.other_domain.module import function`. Never relative imports across domains.
5. **Shared utilities at src/ root** — `routing.py`, `config.py`, constants used by multiple domains. Not in a `utils/` subdirectory.
6. **DOCS.md per domain** — documents modules within that domain (Purpose/Input/Output per module). See documentation rules.

## Module Responsibility

Each module in a domain handles one concern:

| Module Pattern | Responsibility |
|----------------|---------------|
| `search_web.py` | Orchestrates search across engines, deduplicates, formats |
| `scrape_url.py` | Single URL scraping with filtering and garbage detection |
| `browser.py` | Browser lifecycle (start, tab management, cleanup) |
| `rate_limiter.py` | Request throttling per engine |
| `engines/google.py` | Single engine implementation |

A new module is warranted per the complexity thresholds in code-organization.md (>400 LOC, >15 functions, multiple responsibilities).
