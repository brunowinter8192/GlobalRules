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

1. **DB exists:** `docker exec rag-postgres psql -U rag -d rag -c "SELECT 1;"` — if not, create it
2. **GPU servers running:** Check health endpoints before starting long runs
   - Embedding: `curl -s http://localhost:8081/health`
   - SPLADE: `curl -s http://localhost:8083/health`
   - Reranker: `curl -s http://localhost:8082/health`
3. **Report directories:** Scripts create them via `mkdir(parents=True, exist_ok=True)` — but verify after first run

Do NOT let a script crash 3 times on fixable prerequisites. Check once, fix, then run.

## Live System State Claims

When making claims about the CURRENT state of a RAG component (DB contents, server state, config values, indexed data, port open, file presence), verify by running the actual check rather than reading from memory, bead comments, old DOCS, or mental projection.

Sources to mistrust:
- Bead STAND comments — capture session-end state, go stale across sessions
- DOCS sections that haven't been touched in this session — drift constantly with refactors
- Memory of "what was true last week" — same project, different state by today
- Inferences from naming patterns ("collection X is probably the Y data") — verify via one query, don't speculate

Live checks are cheap:
- DB contents (basic): `docker exec rag-postgres psql -U rag -d rag -c "SELECT collection, COUNT(*) FROM documents GROUP BY collection;"`
- DB contents (with vector dims + sparse): `docker exec rag-postgres psql -U rag -d rag -c "SELECT collection, COUNT(*), vector_dims(embedding), sparse_embedding IS NOT NULL FROM documents WHERE collection = '<name>' GROUP BY 1, 3, 4;"` — captures collection size + embedding dimension + sparse-presence in one query
- File presence: `ls <path>` or `find <path> -type f | head` — one tool call
- Config defaults: read the actual file with `Read` or `cat`, don't quote from memory (especially after refactors that move defaults around)
- Server state: `curl http://localhost:<port>/health` or `lsof -i :<port>` — one tool call

Stale info costs: one wrong claim → user correction → trust loss → corrected recap mid-session. Roughly 1 cheap verification call vs. 3-5 correction exchanges.

Test before any state-claim sentence in a Claude reply: "Have I VERIFIED this in THIS session, or am I rendering it from memory?" If the latter, run the check first.

Concrete failures (2026-04-27, single RAG session, three slips):
1. Claimed "RAG_MCP mit ~500 Chunks, searxng mit ~26k" in HNSW explanation as if current state. Reality (live DB query): rag_test contains ONLY RAG_MCP (483 chunks). The "searxng with 26k" was historical state from RAG-7ux March STAND comments, re-rendered as present tense.
2. Claimed `score_threshold` default = 0.5 in eval-coverage discussion. Reality (worker live code inspection): default = 0.0 (filter off). The 0.5 came from an old `src/rag/DOCS.md` note removed during a prior refactor.
3. Wrote `expect: 12` arithmetic in worker prompt verification block. Worker recalculated on the spot: 1 sweep + 11 eval files + 1 baseline = 13. Memory arithmetic vs. actual count.

All three caught (user correction, worker correction, worker correction) before landing in commits. Three live-state slips in one session is a pattern, not noise.

Concrete failure (2026-04-28, RAG-dp3): Bead STAND comment claimed "RAG_MCP in rag_test mit 1024d indexiert (verifiziert via DB-Query)". Actual DB state: 4096d. Made the MRL migration strategy obsolete without bead update. Caught only at plan-mapping; would have been caught upfront with a 5-second `vector_dims()` query. Bead-comment DB-state claims need verification BEFORE being accepted as plan premise.

## Dev Structure Convention

Pipeline modules: `pN_<name>.py` (numbered by dependency order)
Analysis scripts: `A_<name>.py` (produce MD reports to `A_<name>_reports/`)
Dev modules do NOT import from `src/rag/` — they are self-contained.
DB: always `rag` (production database, configured via `.env` `POSTGRES_DB=rag`).
