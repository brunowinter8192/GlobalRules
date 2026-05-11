# searxng — Engine Architecture & Operations

## Multi-Engine Smoke Memory Cap

Multi-engine smoke runs cause severe memory pressure on the shared-Chrome architecture. When `dev/search_pipeline/05_search_smoke.py` runs with 4+ engines × 30 queries: each engine opens its own tab on the shared Chrome instance, ~120 tab-cycles total over ~7-8 minutes wall-clock. Tab cleanup (`tab.close()` in engine `finally` block) does not reliably release Chrome process memory between cycles — observed memory growth approaches multi-GB before completion. With multiple worker tmux sessions each running their own Chrome (preview-worker, mojeek-worker, lobsters-worker spawned in parallel), system memory pressure can trigger macOS pagefile spiral and OOM-killer on tmux server.

**Rule:** before running `05_search_smoke.py` with N≥4 engines:

1. Kill all alive worker tmux sessions you don't need (`worker-cli kill` for any idle worker not currently working). Each worker holds its own Chrome subprocess.
2. Run the smoke from the MAIN project tree (not from a worker), so memory is isolated to one Chrome instance.
3. If multiple workers must remain alive: cap engine count at 3 (Google + DDG + Mojeek), or split the run into 2 batches.
4. Watch Activity Monitor / `vm_stat` during the run. Spike >50% used memory → abort, restart the smoke after killing background workers.

## Engine-Name vs Backend Verification

searxng's engine modules wrap diverse backends — module name is NOT a reliable indicator of the underlying API/scrape strategy. Before claiming "engine X provides a different implementation than Y" within the searxng-engine landscape, READ the source of X. Examples of name-vs-backend mismatch:

- `hackernews.py` wraps the Algolia HN API (same as a hypothetical `hn_algolia.py` would) — not a scrape alternative
- An `openalex.py` module might wrap CrossRef under the hood
- A `mojeek.py` module might use the official Mojeek API instead of scraping

**Rule:** before proposing a searxng engine as alternative implementation strategy: one file read (`gh-cli get_file_content searxng searxng searx/engines/<name>.py`, `cat`, or `grep -l <api_endpoint>`) verifies the backend. Without this, the claim is speculation.
