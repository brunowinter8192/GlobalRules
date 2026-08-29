# dev/ Directory Convention

**dev/ holds development scripts for testing, debugging, and experimentation.**

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

## Output Layout

**A dev script writes its report into `dev/<area>/`.**
- Writing to the console instead is not allowed.
- Inside `dev/<area>/` the report goes to `md/`, `csv/`, or `png/`, chosen by output type.
- The report file carries a descriptive name that traces to its producing script.

**Data outputs stay separate from reports.**
- Scripts also produce data outputs like raw corpora.
   - Data outputs go into their own type-named folder, for example `jsonl/`.
- Data folders never mix into `md/`.

**Reports and data organize by theme in `dev/<area>/`.**
- A dev area and a process-docs area on the same theme share one name.

## Staging

**One-shot scripts live in the worktree or /tmp/ and are never staged.**
- Build forensics and one-shot assertions in the worktree or under /tmp/.
   - Explicitly do not stage them on merge.
- A one-shot assertion that becomes a regression guard folds into an existing dev/ test file.
   - A new file per fix is not allowed.
