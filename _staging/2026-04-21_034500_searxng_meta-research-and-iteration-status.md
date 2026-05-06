# 2026-04-21 — searxng: Meta-Research before Implementation + Iteration Status Communication

## ~/.claude/shared-rules/global/verify-before-execution.md → new section "Meta-Research Before Implementation"

**Meta-Research Before Implementation.** Before committing to an implementation path for a non-trivial problem (anything beyond a ~30min fix with a clearly-working reference in the same repo), spend 15-30min on a targeted external research pass:

1. **GitHub** — search for working implementations of the same problem (pydoll-stealth, puppeteer-extra-plugin-stealth, etc.). Read the top 2-3 repos' READMEs + key source files. Ask: does an out-of-the-box solution exist?
2. **Reddit / forums** — last 3-6 months of posts on the specific detection behavior (e.g. "Google inline consent 2026", "headless Chrome rate limit"). Community-reported recent behavior beats extrapolation from 6-month-old code.
3. **Official docs** — only if relevant (e.g. SerpAPI, Brave Search API) and an alternative approach might obviate the entire custom implementation.
4. **Source-of-truth repo** (pydoll, playwright) — grep for the feature you're about to build. It might already be there.

Output of the research pass: **a 5-line written verdict** stating (a) whether the current plan is the best approach, (b) what alternatives exist, (c) why we stick with or pivot.

**Rule in short:** problem verstehen → schauen wie andere lösen → dann selber ran. Not "problem verstehen → sofort mit eigenem Code loslegen".

**When to skip:** problem is trivially solved by a known-working reference in the same repo (e.g. "port the exact pattern from engine_selectors.py") AND no external factors have changed since the reference was written. Otherwise: research.

Concrete failure (2026-04-21): Google 30/30 session. Jumped straight from "broken parser" to "port existing pattern from engines_eval/" without checking whether there's a pydoll-stealth plugin, whether the community has a better consent-bypass, or whether scraping Google at all is still the right approach in 2026 (vs. SerpAPI, googlesearch-python, etc.). 4 iterations of mühevolle Kleinarbeit. User flagged at session end: "vllt gibt es auf gh und reddit ja ansätze die viel besser und einfacher sind". Would have cost 15-30min upfront to confirm or pivot.

---

**STATUS (2026-05-06 batch4 session):** the other two proposals from this staging have been applied:
- "Per-Iteration User Status" → applied in `opus/workers-2.md` Phase 2
- "Push-Back-Once, Then Dispatch" → applied in `global/communication-2.md` Honest Opinion

The Meta-Research section above is DEFERRED to a separate session per user direction (2026-05-06): "ich finde das eh zu spezifisch. das ist keine project rule das bezieht sich darauf wie man sich zu verhalten hat wenn man einen neuen engine implementiert das macht man nicht dauernd wenn man das projekt betritt. lass das mal beiseite vorerst wir behalten das staging der searxng und entscheiden in einer separaten session darüber." Evaluation pending on whether/where it lands.
