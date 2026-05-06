# Decision Files (decisions/)

Pipeline decision records documenting the rationale for each implementation choice.

## Artifact Density

User-Chat is prose (see `global/chat-output.md`). Everything Claude reads or produces as ARTIFACT — code, DOCS.md, CLAUDE.md, rules/, decisions/, code-comments — is machine-readable and token-dense.

Concrete: tables instead of prose where multiple dimensions need comparison, keywords instead of full sentences, references to `file:line` instead of explanatory paragraphs, no rhetorical filler ("furthermore", "as we can see", "importantly", "it's worth noting"). Where a paragraph IS needed, it is dense — no repetition, no opening sentences that say nothing.

The user reads code and docs through Claude. Token-dense input leaves more context budget for actual work. Every line of prose in a DOCS.md or rule is context cost without information gain over the structured alternative.

Applies to every project, not only Monitor_CC.

## Structure (MANDATORY)

Every decision file has these sections in order:

### Status Quo (IST)
What is CURRENTLY in production code. Code paths, config values, behavior. This section describes reality — not what should be.

### Evidenz
Measurements from dev/ scripts, external research, benchmarks. Data that informs the recommendation. Reference dev/ script paths and result files.

### Recommendation (SOLL)
What the config SHOULD be based on evidence. Three formats:

- **Change:** `X → Y` — Current config should change. State the concrete new value and why.
- **Keep:** `X (no change needed)` — Current config is optimal. One line is enough.
- **Pending:** `Evaluation not done` — No recommendation yet. State what eval is needed. One line is enough.

**Brevity rule:** When SOLL = IST (Keep) or SOLL is unknown (Pending), keep it short — one line per point. Only Change items need detailed justification. This keeps decision files scannable.

When IST ≠ SOLL → migration needed. The diff across ALL decision files = the complete migration plan.

### Offene Fragen
Unanswered questions that affect the recommendation.

### Quellen
RAG Collection references, papers, URLs used as evidence.

## Rules

- **IST reflects production code** — not experiments, not plans, not "what we want"
- **SOLL is evidence-based** — every recommendation cites specific eval results from dev/
- **SOLL only when eval has run** — A full Recommendation (SOLL) section with Change/Keep details is only added after an eval on our data has produced concrete numbers. Files without eval get a minimal SOLL stub: `## Recommendation (SOLL)\nPending — needs evaluation.` This makes the status scannable across all files (which have SOLL, which don't) without adding noise.
- **Migration is deferred** — decision files document WHAT should change, not WHEN. All migrations execute in one batch after complete pipeline eval.
- **Evidenz from sources only** — Evidenz tables and comparisons MUST come from actually read sources (RAG search results, file reads, GitHub API). NEVER present data from training knowledge as Evidenz. If you haven't read the source in this session, it's an ASSUMPTION, not Evidenz. Concrete failure (2026-03-26): Presented a Font-per-Template table from memory without reading any source. User caught it: "was hast du jetzt gerade gelesen?"
