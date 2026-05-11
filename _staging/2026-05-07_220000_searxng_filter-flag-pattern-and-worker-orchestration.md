# 2026-05-07 — searxng: filter-flag pattern + worker orchestration improvements

## ~/.claude/shared-rules/proj_searxng/filter-flag-architecture.md (NEW project rule)

After three iterations (`--books` → `--pdf` → `--docs`, all landed and live-verified 2026-05-07), the filter-flag pattern is stable. Every future `--<class>` filter flag follows this architecture. New file at `~/.claude/shared-rules/proj_searxng/filter-flag-architecture.md`:

```markdown
# searxng — Filter Flag Architecture (--books / --pdf / --docs / future)

## Pattern (locked after 3 iterations)

Every filter flag has these components:

1. **Filter module** at `src/search/<class>_filter.py` (or `<class>_whitelist.py` for whitelist-leaning patterns).
   Three logic styles:
   - **Whitelist + Path-rules + Blacklist override** (--books): inclusion-based, host-set + path-patterns + small host-blacklist for FP-prevention
   - **Hybrid** (--pdf): host-set with subdomain-wildcard + path-patterns + larger host-blacklist
   - **Pure Blacklist** (--docs): exclusion-only, host-set + path-patterns
   Choice driven by empirical probe data: bounded vs unbounded domain space.
   Single export: `is_<class>_url(url: str) -> bool`.

2. **Empirical probe** at `dev/search_pipeline/<NN>_<class>_probe.py` analog `19_books_probe.py` / `20_docs_probe.py`.
   Output report at `01_reports/<class>_probe_<ts>.md`. Required sections:
   - Per-Query URL Listings
   - Global Domain Frequency Table (count ≥ 2)
   - Top-N Inspection (with sample paths)
   - Per-Engine Distribution
   - Run Stats
   For blacklist-driven filters: add Heuristic Coverage Matrix + Miss Set Analysis.

3. **search_web.py wiring** mirroring the established 3 (search book_whitelist/pdf_filter/docs_filter usage):
   - `_<CLASS>_ENGINES` constant — frozenset of engine names that participate
   - `_<CLASS>_MODIFIER` constant — `lambda q: f"{q} <word>"`
   - `<class>: bool = False` param on `search_web_workflow` AND `search_batch_workflow`
   - 3-way+ mutex guard with documented precedence
   - `if <class>:` block: engine-restrict + modifier-set + post-`_merge_and_rank` filter
   - `cache_key(modifier_id="<class>")` — modifier_id param shared with all filter flags

4. **cli.py wiring**:
   - Add to `add_mutually_exclusive_group()` on `search_web` + `search_batch` parsers
   - Independent flag on `search_more` (cache-key matching, NOT mutex)
   - All 3 dispatch blocks pass `<class>=args.<class>` to workflow

5. **SKILL.md update** at `skills/web-research/SKILL.md`:
   - Add row to all 3 parameter tables (search_web, search_batch, search_more)
   - Add "<Class> Lookup Mode" section with 2-3 examples + underfill note + mutex note

6. **Live test** — at minimum 3 queries via prod CLI from worktree before merge.

## Engine selection patterns

| Filter | Engines (likely) | Rationale |
|--------|------------------|-----------|
| --books | google + ddg + mojeek (general only) | Books are commercial-marketplace-focused, academic engines bring no value |
| --pdf | google + ddg + mojeek + scholar (general + scholar) | PDFs come from both general (.pdf URLs, university servers) and academic (Scholar TIER1 transforms) |
| --docs | google + ddg + mojeek (general only) | Docs are general-web content; academic indexes don't carry vendor docs |

Drop low-yield engines per filter via empirical probe data — e.g. openalex/crossref dropped from --pdf because they only return doi.org URLs that don't pass is_pdf_url.

## Underfill is accepted

Filter mode + slot allocation = potential underfill (some queries return <20 URLs). This is documented behavior. Pooling-rethink (bead g82) will address cross-flag with fresh allocation logic post-RRF-adoption.
```

---

## ~/.claude/shared-rules/opus/workers-2.md → "Timer & Polling Flow" section

Append a new subsection after the timer/polling flow:

```markdown
### Test-Environment-Artifact Debug Spirals — Interrupt Rule

When a worker is in Phase B+C and live tests show failures, classify the failure:
1. **Engine/code bug** — engine logic is wrong, snippet selector wrong, etc. → worker debugs, OK
2. **Test-environment artifact** — failure caused by test setup, not code (rate-limit from Phase A burst hitting same IP, stale cache, env var missing, etc.) → worker should NOT debug deeper

If worker has been investigating ONE issue for ≥10 minutes AND captured-pane shows they're reading rate_limiter / probe-session / env-var related code, OPUS MUST interrupt:

```
worker-cli send <name> "Stop the [issue] investigation. The Live-Test failure is a test-environment artifact (probe burned through bucket / IP rate-limit / etc.) — that's not an engine bug. Engine code is the deliverable.

Wrap up NOW: commit current state, output Completion Checklist with the artifact noted as a follow-up observation, do NOT debug further."
```

The signal: a worker debugging "why is this empty in my live test" when the engine has already proven correct in Phase A live extraction is wasting context on a test-environment problem.

Verified 2026-05-07: semscholar worker spent 14 min in rate-limiter investigation (37% → 26% context drop) before Opus intervened. Worker had already validated engine in Phase A, smoke ran 21/30 OK. The 5 EMPTY live results were probe-session IP rate-limit carryover. Wrap-up message reduced waste from 26%→ commit clean.
```

---

## ~/.claude/shared-rules/worker/worker-rules.md → ceiling-probe section

Add new subsection (insert after existing probe rules):

```markdown
### Ceiling Probes — test boundaries, not adjacent

When probing per-engine result-count ceiling, test ACROSS the expected range to find the actual cap, not just adjacent values that all pass.

**Wrong:** test page=1, page=2, page=3 → all return 10 → conclude "ceiling 30, max-results-per-call hardcoded by SS UI"

**Right:** test page=1, 5, 10, 20, 50, 100 → find where results stop returning OR site shows "no more results" page

The boundary is what matters for the engine's `ENGINE_MAX_RESULTS` setting. "Confirmed at least 30" is not the same as "max is N" — and follow-up sessions need the actual N to make pooling decisions.

When the engine is browser-rendered SPA (results populated client-side via JS), curl-based probing returns empty. Use the actual browser/pydoll probe (or the engine's own internal request mechanism) to test high page numbers. Document the actual cap or document explicitly that "max not probed beyond N pages — would need browser-side probe at higher page numbers".

Verified 2026-05-07: SS Phase A reported "ceiling 10/page hardcoded" but only tested pages 1-3. User question "was ist denn das maximum?" had no empirical answer; required follow-up curl probe (which failed due to SPA rendering) and ultimately admission that real max wasn't probed.
```

---

## Why these are project-rule / worker-rule / opus-rule, not global

- **filter-flag-architecture.md** — searxng-specific feature pattern, no carryover to other projects
- **workers-2 interrupt rule** — universal Opus orchestration behavior across projects (any project can have test-env artifacts)
- **worker-rules ceiling-probe** — universal worker behavior (probes happen across projects)
