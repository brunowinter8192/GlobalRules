# Server Operations

**MCP Server** starts automatically via `mcp-start.sh` (PostgreSQL + FastMCP). Supports `list_collections`, `list_documents`, `read_document` without GPU servers.

**GPU Servers** auto-start on demand via `server_manager.py` when search/index operations are called. Auto-stop after 15 minutes idle (`IDLE_TIMEOUT=900`, env `RAG_SERVER_IDLE_TIMEOUT`). Manual control:

```bash
./venv/bin/python workflow.py server status           # Check all servers
./venv/bin/python workflow.py server start             # Start all
./venv/bin/python workflow.py server stop              # Stop all
./venv/bin/python workflow.py server restart splade    # Restart one
./start.sh                                            # PostgreSQL + all GPU servers
```

Server configs (ports, model paths, flags) are defined in `src/rag/server_manager.py` — single source of truth.
