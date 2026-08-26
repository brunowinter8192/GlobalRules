# dev/ Directory Convention

**dev/ holds development scripts for testing, debugging, and experimentation.**

## Purpose — Analysis Only

**Three activities stay separate.**

| Activity | Handling |
|---|---|
| Live-verification of a source fix | Build the fix in the worktree. Opus or the user restarts the app, triggers the scenario, and observes. This is the standard loop and needs no script. |
| Forensics on existing data | The scenario cannot be triggered live without touching the environment. Load the real data source, call the function with a real entry, and measure. The script lives in the worktree or /tmp/ and is gone once the root cause is clear. |
| Assertion across many data points after a fix | An invariant over N entries is infeasible by hand. With permanent regression value, fold the case into an existing dev/ test file. With one-fix value, the script lives in the worktree or /tmp/ and is gone on merge. |

**Permanent value decides what earns a place in dev/.**

| Permanent value, so dev/ | One-shot value, so worktree or /tmp/ |
|---|---|
| Benchmarks, evals, investigation modules for a documented problem, and growing assertion suites | Session forensics scripts and one-shot fix verifiers without regression value |

**The deciding question is whether the script is useful to another agent with zero context.**
- If yes, it belongs in dev/.
- If no, it belongs in the worktree or /tmp/.
- dev/ hands a zero-context agent which tests ran, when, how, and with what result.
   - The timeframe is irrelevant for that.

| Example | Verdict |
|---|---|
| `dev/retrieval/run_smoke.py` | Permanent analysis tool, so dev/. |
| `dev/retrieval/test_assertions.py` | Growing assertion library with new cases per fix, so dev/. |
| `dev/feature_debug/` | Investigation module for a documented problem, so dev/. |
| `verify_fix_works.py` | One-shot, so worktree or /tmp/. |

**One-shot scripts live in the worktree or /tmp/ and are never staged.**
- Build forensics and one-shot assertions in the worktree or under /tmp/.
   - Explicitly do not stage them on merge.
- A one-shot assertion that becomes a regression guard folds into an existing dev/ test file.
   - A new file per fix is not allowed.
