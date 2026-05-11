# 2026-05-11 — Monitor_CC: RAG dual-flow indexing scoping

## ~/.claude/shared-rules/proj_Monitor_CC/rag-indexing.md (NEW file)

When the user asks to "index docs" or "index sources" for Monitor_CC, FIRST distinguish where the content belongs before scoping a worker or writing the manifest.

**Project-local pipeline** (`Monitor_CC-meta`, `Monitor_CC-features`): Files live IN the Monitor_CC repo. Managed via `.rag-docs.json` + `rag-cli update_docs .`. Hash-synced per `(collection, relative_path)`. Covers DOCS.md, decisions/*.md, decisions/OldThemes/, sources/sources.md. NOT external reference material.

**Central reference pipeline** (`Monitor_reference`): Files live in `/Users/brunowinter2000/Documents/ai/Meta/ClaudeCode/MCP/RAG/data/documents/Monitor_reference/`. Managed via `python workflow.py index-dir --input <path>` (note: `--input` flag required, NOT positional). `data/` is gitignored in the RAG repo — files are local-only artifacts. Used for generic Anthropic API doc mirrors, vendor docs, papers, anything without specific per-file decision relevance.

**Decision rule:** if the source is a generic external reference whose per-file decision-mapping cannot be reconstructed, it belongs in the central pipeline. If it's project-internal research, an architectural decision, a per-feature deep-dive that maps to specific bead/pipeline-step, it belongs project-local under `decisions/` or `decisions/OldThemes/`.

**sources.md status convention:**
- Files in `Monitor_reference/`: status `Indexed (RAG: Monitor_reference)`, Decision Steps `misc` (no per-file mapping maintained)
- External URLs/repos not mirrored: `Referenced` or `Verified`
- Forum sources (Reddit/HN/LinkedIn from plugin searches): permanent `Referenced`, never RAG-indexed

**Anti-pattern (session 2026-05-11):** spawning a worker to "index sources" without first checking whether the sources are project-local content or external reference material. Worker followed instructions and grouped 92 Anthropic API mirrors into `Monitor_CC/sources/<topic>/` subfolders, declared a `Monitor_CC_reference` glob in `.rag-docs.json`. Architecturally wrong — those files belonged in `Monitor_reference/` (central), not Monitor_CC's repo. Worker's commit had to be discarded, files moved cross-repo, sources.md rewritten.

**Prompt template addendum for indexing tasks:** include explicit verification step: "Before proposing folder structure, classify each file: (a) project-local — has clear decision/feature relevance → stays in repo; (b) generic reference — no per-file decision mapping → goes to `/Users/.../Meta/ClaudeCode/MCP/RAG/data/documents/<collection>/`. Report classification ratio in Phase A before proceeding."
