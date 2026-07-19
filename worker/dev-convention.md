# dev/ Directory Convention

Development scripts for testing, debugging, and experimentation.

## Purpose — Analysis Only, Not Debug-Script Dumping

Three activities, kept separate:

| Activity | Handling |
|---|---|
| **Live-verification of a source fix** | Build the fix in the worktree; Opus or user restarts the app, triggers the scenario, observes. Standard loop — no script. |
| **Forensics on existing data** | Can't be triggered live without touching the environment. Load the real data source (JSONL, log), call the function with a real entry, measure. Script in worktree or `/tmp/`, throwaway — gone once root cause is clear. |
| **Assertion across many data points after a fix** | Invariant over N entries, infeasible by hand. Permanent regression-guard value → fold the case into an EXISTING `dev/` test file. One-fix value only → worktree or `/tmp/`, gone on merge. |

What earns a place in `dev/` vs what does not:

| Permanent value → `dev/` | One-shot value → worktree / `/tmp/` |
|---|---|
| Benchmarks, evals, investigation modules for a documented problem, growing unit-test / assertion suites | Session forensics scripts, one-shot fix verifiers (no regression value) |

**Rule of thumb — useful to another agent with zero context?**
Yes → `dev/`. No → worktree or `/tmp/`. dev/ exists to hand a zero-context agent which tests ran, when, how, and with what result — the timeframe is irrelevant.

| Example | Verdict |
|---|---|
| `dev/retrieval/run_smoke.py` | permanent analysis tool ✅ |
| `dev/retrieval/test_assertions.py` | growing assertion library, new cases per fix ✅ |
| `dev/feature_debug/` | investigation module for documented problem ✅ |
| `verify_fix_works.py` | one-shot → worktree or `/tmp/` ❌ |

**Consequence — one-shot scripts live in the worktree or `/tmp/`, never staged.**
When forensics or one-shot assertion is needed, you build the script in the worktree (not staged on merge — explicitly do not stage) or under `/tmp/`. When a one-shot assertion becomes a permanent regression guard, the test case is folded into an EXISTING `test_*.py` in `dev/` — no new file per fix.
