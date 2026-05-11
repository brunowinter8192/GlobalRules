# 2026-05-06 — SEARXNG: scrape cleanup pipeline + overview mode design

## ~/.claude/shared-rules/global/verify-before-execution.md → new section near "Numeric Values in Reports"

**Asymmetric User Preferences Trap (Metric Symmetry):** When a user states a preference asymmetrically — "I prefer X over Y", "lieber A als B", "X ist mir wichtiger als Y" — symmetric metrics (F1, similarity ratio, accuracy) penalize both sides of the trade-off equally and produce recommendations that don't match the asymmetric preference.

Before designing or applying a metric to compare candidates: explicitly ask whether the metric is symmetric, and whether the user's stated preference is asymmetric. If both are true, the metric is wrong for the use case. Either:
- Pick an asymmetric metric (e.g. precision when noise-removal > content-loss; recall when content-preservation > noise-tolerance)
- Add hard constraints + soft optimization (e.g. "title MUST be present" + "minimize chrome lines")
- Stratify reporting so the asymmetric trade-off is visible per axis

Concrete failure (2026-05-06, searxng overview-mode): user stated "lieber strippe ich noise auch wenn dafür weniger content übrig bleibt — quality > quantity". Opus designed line-set F1 metric (recall + precision averaged) to compare 36 filter configs against clean-raw baseline. Top F1 config (`none + cleaned_html`, F1=0.98) was recommended as best, then revised to `prune_030 + cleaned_html` (F1=0.89, mild compromise). User pushed back with concrete URL example showing `prune_048` (F1=0.75, current prod) was the actually-preferred output because Skip-to-Content + nav was stripped — even though F1 was lower because real content was also stripped. F1 punishes content-loss as much as chrome-retention; user only cares about chrome-retention. The same sweep data, read with precision-weighted lens, would have surfaced prune_048 as correct from the start. Roughly 4-5 exchanges spent revising recommendations before user explicitly named the asymmetric mismatch.

## ~/.claude/shared-rules/global/verify-before-execution.md → new section near "Tool Extension"

**Library-First Principle (Built-Ins Before Custom Mechanism):** When working with an established third-party library (web framework, scraping library, ML library, etc.) and proposing a new mechanism (filter, transformation, processor), EXHAUSTIVELY enumerate the library's built-in features for that concern BEFORE proposing custom code.

Concrete process:
1. List the library's documented public API surfaces relevant to the concern (e.g. for content extraction: PruningContentFilter, BM25ContentFilter, LLMContentFilter, content_source options, excluded_selector, fit_markdown vs raw_markdown variants).
2. State which of those have already been tried in this project's history (check existing decision files / past code).
3. State which of those have NOT been tried, and why they're being skipped (cost, scope, known incompatibility — with concrete reason).
4. Only then propose custom mechanism, framed explicitly as "library doesn't cover X because Y, so custom code Z fills the gap".

Skipping this step produces architectural drift toward "we built a worse version of what the library already provides", and forces user pushback to redirect.

Concrete failure (2026-05-06, searxng overview-mode): Opus proposed "Shape B — cleaned-raw + structural truncation" as Overview Mode architecture, including a custom truncation script with section-boundary cuts. User pushed back: "hat crawl4ai nicht schon built in filter? das was wir aktuell machen. crawl4ai hat doch schon library intern alle funktionen die wir brauchen. 0.48 pruning war damals empirisch validiert." Crawl4AI's PruningContentFilter / BM25ContentFilter / content_source options had been in the prod code since March 2026 — Opus knew this from reading scrape_url.py earlier in the same session, but mentally bracketed them as "the heuristic that broke chuniversiteit table" and proposed custom-truncation as alternative. The right pattern was: enumerate Crawl4AI's filter options exhaustively, find the trade-off the user wanted (PruningFilter @ 0.48), then propose tuning of existing dial — not new dial. ~3 exchanges wasted on the wrong architectural framing.

Both lessons compound: the library-first miss caused us to build a custom comparison framework (sweep + diff analyzer ~700 LOC). The work itself was useful (validated prod's existing config empirically). But the underlying decision — "use Crawl4AI's filter, tune its parameter" — was already obvious from reading the existing decision file. Library-first would have shortened the path.
