# 2026-05-06 — SEARXNG: NULL embedding root-cause investigation

Session walked through the full RAG pipeline for the new Crawl4AI_Docs collection (cleanup-and-index workflow), discovered a 26-chunk silent loss, went down the wrong "server-state degradation" hypothesis for ~30 minutes before finding an existing decisions/ entry that documented the actual cause (Qwen3 tokenizer prefix). Lessons below.

## ~/.claude/shared-rules/global/verify-before-execution.md → "Root Cause Before Fix" or new sub-section

When investigating a bug in a project component, grep `<project>/decisions/` for keywords matching the symptom BEFORE forming new hypotheses. Decisions docs are the project's accumulated empirical findings — if the bug already happened once, the diagnosis lives there. Skip this and you reproduce the investigation from scratch.

Concrete failure (2026-05-06, RAG NULL embeddings): observed 26 chunks dropped silently during index-dir. Spent ~30 min building a "llama-server state degradation over uptime" hypothesis with multiple kill-restart probes. The actual cause was already in `RAG/decisions/index02_dense_embedding.md:28`: "Known Issue — NULL Embeddings: llama-server returns NULL vectors for chunks starting with naked `import` statements ... Fix: Prefix every chunk with `search_document: ` before embedding. Validated." Two-minute grep would have surfaced this. The "state degradation" framing was wrong — content-deterministic tokenizer issue, identical 26 chunks fail every run.

Test: before forming the second hypothesis on a bug, did you already grep the decisions/ folder for matching keywords (the symptom, the affected component, the model name)? If no → grep first.

## ~/.claude/shared-rules/global/verify-before-execution.md → "Trust-but-Verify Decision Docs"

When a `decisions/` entry claims a fix is "validated", "applied", or "in production", grep the actual production call site to verify the fix is wired up — do NOT trust the claim alone. Decision files describe SOLL state; if the fix was validated standalone but never integrated into the production code path, the doc lies until somebody notices.

Concrete failure (2026-05-06, RAG): `decisions/index02_dense_embedding.md` documented the `search_document: ` prefix fix as "Validated: all NULL chunks produce valid embeddings with prefix." Production code in `src/rag/indexer.py:parallel_embed` called `embed_workflow(texts)` WITHOUT the prefix — the fix was never integrated. ~3-4% silent chunk loss continued for the lifetime of the indexer, with the decision doc claiming the issue was solved. Verification step: when reading a "validated" claim, grep the call site (`grep -n "embed_workflow" src/`) before concluding the fix is live.

## ~/.claude/shared-rules/global/tool-use.md → "Bash CLI" or "Hard Rules" subsection

`git add -A` is forbidden when committing a single-purpose change. Always specify file paths explicitly. `-A` sweeps in unrelated tracked-modifications and untracked files (build artifacts, beads backup, leftover decisions drafts) and contaminates an otherwise clean commit. The git-check tool's automatic staging is fine because it has skip patterns; manual `-A` doesn't.

Concrete failure (2026-05-06, github plugin commit): used `git add -A` for a 1-line skill frontmatter cleanup. Commit got 6 files, 316 insertions (a 304-line `decisions/OldThemes/research_tool_wishlist_*.md` plus 4 `.beads/backup/*` auto-generated files). `git reset HEAD~1` + explicit `git add skills/github-search/SKILL.md` + re-commit was needed. The 1-line edit ended up correct as commit `f514c1c` (1 file, 1 deletion) but burned 3 extra Bash calls and polluted the diff.

Rule: `git -C <repo> add <explicit/path>` only. `-A` is reserved for empty-tree initial commits or full-clean bulk operations explicitly intended.

## ~/.claude/shared-rules/global/tool-use.md → "Python: heredoc for one-shot, Write + Exec for iteration" — generalize Background pattern

Add to existing rule: When a Python probe runs longer than ~20 seconds (embedder probes, indexing tests, multi-batch loops, anything calling external services with N>10 sequential requests), use the same background+unbuffered pattern as the cleanup-and-index Phase 2:

```bash
PYTHONUNBUFFERED=1 ./venv/bin/python3 /tmp/<probe_name>.py > /tmp/<probe_name>.log 2>&1 &
```

Status probe via `tail /tmp/<probe_name>.log` — the unbuffered redirect makes the log line-by-line readable mid-run.

Foreground for long probes blocks the agent from responding to user mid-investigation. Background without `PYTHONUNBUFFERED=1` is equally bad — log file stays empty until process exit due to Python full-buffering, defeating the whole purpose. Both must be set.

Concrete failure (2026-05-06, embedding probe): wrote a 22-batch indexing reproduction probe (~12 min wall-clock). First attempt: ran foreground inside Bash heredoc, blocking. Second attempt: ran background but without PYTHONUNBUFFERED — log stayed at `Total chunks: 704` for 12 minutes despite probe actively running. User had to point out twice that the cleanup-and-index Phase 2 pattern (which I had JUST WRITTEN to fix this exact problem in production indexing) applies to my own diagnostic Python work too. Third attempt with both set worked correctly.

Test: is the Python script likely to take >20s? Add `PYTHONUNBUFFERED=1` and `&` before running.
