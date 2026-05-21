# dev/ Directory Convention

Development scripts for testing, debugging, and experimentation.

## dev/ Purpose — Analysis Only, Not Debug-Script Dumping

Three different activities get easily mixed in debugging — this rule separates them:

1. **Live-Verification of source-code fix.** Worker builds fix in worktree, Opus or user restarts the application, triggers the scenario, observes result. Standard loop, no script.

2. **Forensics on existing data.** Example: "does this code line wrap after `truncate_visible` or not?" — can't be triggered live without manipulating the live environment. Load the real data source (JSONL, log), call the function with a real entry, measure properties. Without such a script you speculate in circles. Script lives in the worker worktree or `/tmp/`, throwaway — once the analysis clarifies root cause, the script is worthless.

3. **Assertion across many data points after a fix.** Example: pair-alignment in `render_messages` across 145 stripped entries. Live-test would mean 145 manual expand-clicks, infeasible. Script does it in milliseconds. Decision point: does the assertion have permanent regression-guard value? Then fold the test case into an EXISTING test file in `dev/`. Just historical value for the one fix? Then worktree or `/tmp/`, gone on merge.

**What goes in `dev/` (permanent value):**
- Benchmarks (pipeline performance measurements)
- Evals (retrieval/reranker/quality evaluation suites)
- Investigation modules for a documented problem (see `documentation.md` → "dev/ Investigation Modules")
- Growing unit-test suites / assertion libraries

**What does NOT go in `dev/` (one-shot value):**
- Forensics scripts of a session (Category 2)
- One-shot fix verifiers (Category 3 without regression value)

**Rule of thumb:** "is this script still useful in 3 months?" Yes → `dev/`. No → worktree or `/tmp/`.

**Examples:**
- `dev/pipeline/run_smoke.py` → permanent analysis tool ✅
- `dev/pipeline/test_assertions.py` → growing assertion library, new test cases come per fix ✅
- `dev/feature_debug/` → investigation module for documented problem ✅
- `verify_fix_works.py` → one-shot, does NOT belong in `dev/`. Worktree or `/tmp/` ❌

**Worker consequence:** when forensics or one-shot assertion is needed, the worker builds the script in the worktree (not staged on merge — explicitly do not stage) or under `/tmp/`. When a one-shot assertion becomes a permanent regression guard, the test case is folded into an EXISTING `test_*.py` in `dev/` — no new file per fix.

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
2. **DOCS.md per pipeline stage** — pipeline dirs with multiple scripts MUST have a DOCS.md. Single-script pipeline dirs are documented in the parent DOCS.md (per `global/documentation.md` §dev/ Suites).
3. **`pN_` prefix for pipeline modules** — numbered by position/dependency order within the directory. Self-contained, no imports from `src/`. These ARE the dev implementations that get migrated to prod when proven.
4. **`A_` prefix for analysis/eval scripts** — import from `pN_` modules, produce MD reports. Output to `A_<name>_reports/`.
5. **Dev is self-contained** — dev modules do NOT import from `src/`. Dev mirrors prod interfaces but is independent. When a dev implementation is proven, it gets migrated to `src/` (lean, without report output).
6. **Renumber when structure changes** — numbering is per-directory. Adding/removing modules = renumber the directory.
7. **Reports include timestamps** — output filenames contain `<label>_<timestamp>` for history tracking
8. **cleanup/** — utility scripts without decision mapping (data cleanup, migration). No pipeline grouping needed.
9. **MD output, never console** — dev scripts write results to MD files in their report directories. Console output is limited to the report file path. Analysis happens by reading the MD together, not by dumping into the terminal.
10. **Python execution** — ALL Python commands MUST use `./venv/bin/python` (not `python` or `python3`). The system Python does not have project dependencies installed.
11. **Rate limiting** — when a suite script makes multiple HTTP requests to external services, include a 1-2s delay between requests to avoid triggering rate limits or engine suspensions.
