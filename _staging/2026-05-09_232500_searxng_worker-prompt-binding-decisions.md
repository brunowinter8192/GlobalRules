# 2026-05-09 — searxng: Worker prompt formulation, encoding user-binding decisions

## opus/opus-workers-1.md → Phase 1 Prompt Structure → MUST NOT include

Add bullet under `MUST NOT include`:

- Options for design decisions the user has already explicitly made during PLAN. When the user has set a binding design call (e.g. "no IDF", "no engine grouping", "drop class buckets"), the worker prompt MUST encode that choice as a constraint, NOT as one of N options to pick. Offering the library default as an option when the user said "different from library default" is a verification escape — the worker reasonably picks the mechanically-simplest option, which violates user intent. Cost: one round-trip to correct in Phase A evaluation.

**Test before submitting prompt:** for every "pick one of {A, B, C}" formulation in the prompt, check whether the user's most recent binding decision unambiguously narrows to one of these. If yes → encode as constraint, drop the option list. If unclear → ask user to clarify before dispatch, do not delegate the disambiguation to the worker.

## Trigger

2026-05-09 searxng-g82 BM25 sweep prompt (`/tmp/spawn-worker-searxng-bm25-sweep.md`) offered IDF as a 3-way option `{skip entirely, per-pool IDF, uniform IDF=1}` when the user's earlier explicit call was "no IDF — stopword filter is enough". Worker picked per-pool IDF (BM25Okapi library default, simplest implementation) — violated user intent. Detected at Phase A report; corrected via `worker-cli send` round-trip to switch to uniform IDF via BM25Uniform subclass override of `_calc_idf`. Net cost: one round-trip + the worker's wasted Phase-A planning effort on a contradicted option.

**Pattern:** user says "X is not what we want", Opus distills into worker prompt as "pick one of {X, Y, Z}". Loose distillation of binding constraints into option lists.

## Cross-reference

Existing rule `~/.claude/shared-rules/global/communication-2.md` → "User-Driven Pivots are Binding" covers Opus's chat recommendations after a user pivot. This staging extends the same principle to worker prompt formulation — prompts are recommendations to a delegate, the same binding-respect logic applies.
