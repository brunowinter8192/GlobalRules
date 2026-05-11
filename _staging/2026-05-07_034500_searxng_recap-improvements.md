# 2026-05-07 — searxng: empirical-investigation session improvements

## ~/.claude/shared-rules/global/communication-2.md → "Honest Opinion / Options Require Recommendation" section

Add Recommendation-Discipline subsection:

When making a technical recommendation (which value to pick, which approach to take, which engine to use), the justification MUST be empirical (measured data this session or referenced from prior measurement) or mechanistic (causal explanation of WHY the recommendation works), not merely descriptive (stating that X is N-times bigger than Y, or that X "fits" without explaining the mechanism).

A description of magnitude is not a justification of choice. "Value V is N times the current value" tells the reader the size of the change but not why V is the right value. The actual justification needs to either reference a measurement (V was the empirical sweet spot at level X, derived from data Y) or a mechanism (V works because mechanism Z which the smaller alternatives lack).

Test before publishing a recommendation: can the user reasonably ask "why this value?" and the answer is contained within the recommendation itself, or does answering require an additional explanation about mechanism or evidence? If additional explanation is needed — add it BEFORE publishing the recommendation, not after the user pushes back.

## ~/.claude/shared-rules/global/verify-before-execution.md → "Numeric Values" section extension

Extend the "Numeric Values in Reports / Analysis / Chat" rule to cover structural claims about code: number of selectors that match, presence/absence of comments in code, defaults of constants, presence/absence of decision-file documentation.

When stating in chat that "engine X returns exactly N results" or "there is no comment in the code about Y" or "decisions/ has no entry on Z", treat these as structural claims about code that require the same verification standard as numeric values. Either: read the source in this session and quote the location (file:line, decision-file path), or explicitly label as "vermute X, müsste verifiziert werden".

The risk pattern: structural claims sound declarative ("X is the case") rather than estimative ("I think X"), so they're trusted more than they should be when stated without source. The fix is the same as for numeric claims — verify or label as unverified.
