# dev/ Directory Convention

Development scripts for testing, debugging, and experimentation.

## Structure

```
dev/
├── <pipeline_stage>/              # Grouped by pipeline stage
│   ├── DOCS.md                    # MANDATORY — describes modules and scripts
│   ├── p1_<first_module>.py       # Pipeline module, numbered by position/dependency
│   ├── p2_<second_module>.py
│   ├── A_<analysis_script>.py     # Analysis/eval script
│   ├── A_<name>_reports/          # Output dir for analysis script
│   └── ...
└── cleanup/                       # Utility scripts (no pipeline mapping)
```

## Naming Convention

**Pipeline modules (`pN_`):** Numbered by position in the pipeline or dependency order. These are self-contained implementations that can be migrated to prod (`src/`) when proven. `p1_` runs first or has no dependencies, `p2_` depends on `p1_`, etc.

**Analysis scripts (`A_`):** Scripts that evaluate, benchmark, or analyze the pipeline modules. They import from `pN_` modules and produce MD reports. Reports go to `A_<name>_reports/`.

**Numbering is per-directory** — each pipeline stage dir starts at `p1_`. When modules are added or removed, renumber within that directory.

**Example (RAG):**
```
dev/
├── indexing/
│   ├── p1_chunker.py              # Pipe step 1: text → chunks
│   ├── p2_embedder.py             # Pipe step 2: chunks → dense embeddings
│   ├── p3_sparse_embedder.py      # Pipe step 3: chunks → sparse embeddings
│   ├── p4_db.py                   # Pipe step 4: storage + search
│   ├── p5_indexer.py              # Pipe step 5: orchestrates 1-4
│   ├── A_index_collection.py      # Analysis: index + report stats
│   ├── A_index_collection_reports/
│   ├── A_chunking_stats.py        # Analysis: chunk size distribution
│   └── A_chunking_stats_reports/
├── retrieval/
│   ├── p1_retriever.py            # Pipe step 1: query → results
│   ├── A_retrieval_sandbox.py     # Analysis: test queries across modes
│   └── A_retrieval_sandbox_reports/
└── cleanup/
```

## Rules

1. **Pipeline grouping** — top-level dev/ dirs correspond to pipeline stages (e.g., `indexing/`, `retrieval/`)
2. **DOCS.md per pipeline stage** — every pipeline dir MUST have a DOCS.md describing its modules and scripts
3. **`pN_` prefix for pipeline modules** — numbered by position/dependency order within the directory. Self-contained, no imports from `src/`. These ARE the dev implementations that get migrated to prod when proven.
4. **`A_` prefix for analysis/eval scripts** — import from `pN_` modules, produce MD reports. Output to `A_<name>_reports/`.
5. **Dev is self-contained** — dev modules do NOT import from `src/`. Dev mirrors prod interfaces but is independent. When a dev implementation is proven, it gets migrated to `src/` (lean, without report output).
6. **Renumber when structure changes** — numbering is per-directory. Adding/removing modules = renumber the directory.
7. **Reports include timestamps** — output filenames contain `<label>_<timestamp>` for history tracking
8. **cleanup/** — utility scripts without decision mapping (data cleanup, migration). No pipeline grouping needed.
9. **MD output, never console** — dev scripts write results to MD files in their report directories. Console output is limited to the report file path. Analysis happens by reading the MD together, not by dumping into the terminal.
10. **Python execution** — ALL Python commands MUST use `./venv/bin/python` (not `python` or `python3`). The system Python does not have project dependencies installed.
11. **Rate limiting** — when a suite script makes multiple HTTP requests to external services, include a 1-2s delay between requests to avoid triggering rate limits or engine suspensions.

## Library Debugging

When scraping or content extraction has issues:
1. Read `requirements.txt` to identify which library handles the functionality
2. Look up the library's GitHub repo — read source code, issues, docs
3. Build the fix based on actual library API, not assumptions
4. Test with a debug script in `dev/` before modifying production code
