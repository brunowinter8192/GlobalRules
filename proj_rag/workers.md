# Project Worker Constraints

**No GPU/embedding operations in workers.** Workers must NOT run scripts that embed large corpus data (embedding 2337 chunks takes 30+ min and blocks GPU). All embedding, indexing, and eval sweeps run in the main session (background commands). Workers do: code analysis, docs, research, writing plans.

**Debug/Exploration workers (MCP projects):** MUST write exploration scripts, NOT call MCP tools (can't share browser session).
