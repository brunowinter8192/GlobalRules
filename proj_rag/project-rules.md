# RAG Project Rules (Opus + Worker)

## Post-Merge Cleanup

After `worker_merge`: verify that deleted files/directories are actually gone in the main branch. Worktree deletes don't always propagate cleanly.

```bash
# After merge, check for orphaned dirs
ls dev/indexing/  # should only have pN_ modules, A_ scripts, DOCS.md, report dirs
ls dev/retrieval/ # same pattern
```

If old directories remain: delete them manually in the main branch.

## Dev Script Prerequisites

Before running any dev script that writes to the database:

1. **DB exists:** `docker exec rag-postgres psql -U rag -d rag_test -c "SELECT 1;"` — if not, create it
2. **GPU servers running:** Check health endpoints before starting long runs
   - Embedding: `curl -s http://localhost:8081/health`
   - SPLADE: `curl -s http://localhost:8083/health`
   - Reranker: `curl -s http://localhost:8082/health`
3. **Report directories:** Scripts create them via `mkdir(parents=True, exist_ok=True)` — but verify after first run

Do NOT let a script crash 3 times on fixable prerequisites. Check once, fix, then run.

## Dev Structure Convention

Pipeline modules: `pN_<name>.py` (numbered by dependency order)
Analysis scripts: `A_<name>.py` (produce MD reports to `A_<name>_reports/`)
Dev modules do NOT import from `src/rag/` — they are self-contained.
DB: always `rag_test`, never `rag`.
