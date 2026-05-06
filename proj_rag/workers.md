# Project Worker Constraints

**No GPU/embedding operations in workers.** Workers must NOT run scripts that embed large corpus data (embedding 2337 chunks takes 30+ min and blocks GPU). All embedding, indexing, and eval sweeps run in the main session (background commands). Workers do: code analysis, docs, research, writing plans.

**Debug/Exploration workers (MCP projects):** MUST write exploration scripts, NOT call MCP tools (can't share browser session).

## RAG Data Output Paths (CRITICAL)

When writing to `data/documents/` (PDF conversions, chunks, JSON):

- **ALWAYS** write to the RAG project path: `~/Documents/ai/Meta/ClaudeCode/MCP/RAG/data/documents/<collection>/`
- **NEVER** write to the plugin cache path: `~/.claude/plugins/cache/.../rag/<version>/data/documents/`
- The plugin cache is a COPY of source code — `plugin-sync.sh` overwrites it. Files written there are LOST.
- The RAG project repo is the persistent storage. The MCP server reads from there.

Concrete failure (2026-03-17): Worker wrote 700 chunks (5 converted PDFs) to plugin cache path. All lost after session. 2+ hours of MinerU conversion + LLM cleanup wasted.
