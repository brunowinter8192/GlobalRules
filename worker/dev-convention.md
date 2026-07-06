# dev/ Directory Convention

Development scripts for testing, debugging, and experimentation.

## dev/ Purpose — Analysis Only, Not Debug-Script Dumping

Three activities, kept separate:

1. **Live-Verification of source-code fix.** You build the fix in the worktree, Opus or user restarts the application, triggers the scenario, observes result. Standard loop, no script.

2. **Forensics on existing data.** Example: "does the output wrap at function X or not?" — can't be triggered live without manipulating the live environment. Load the real data source (JSONL, log), call the function with a real entry, measure properties. Script lives in your worktree or `/tmp/`, throwaway — gone once the analysis clarifies root cause.

3. **Assertion across many data points after a fix.** Example: an invariant checked across N entries where live-testing would mean N manual steps — infeasible by hand, milliseconds by script. Decision point: does the assertion have permanent regression-guard value? Then fold the test case into an EXISTING test file in `dev/`. Just historical value for the one fix? Then worktree or `/tmp/`, gone on merge.

**What goes in `dev/` (permanent value):**
- Benchmarks (performance measurements)
- Evals (retrieval/reranker/quality evaluation suites)
- Investigation modules for a documented problem
- Growing unit-test suites / assertion libraries

**What does NOT go in `dev/` (one-shot value):**
- Forensics scripts of a session (Category 2)
- One-shot fix verifiers (Category 3 without regression value)

**Rule of thumb:** "is this script still useful in 3 months?" Yes → `dev/`. No → worktree or `/tmp/`.

**Examples:**
- `dev/retrieval/run_smoke.py` → permanent analysis tool ✅
- `dev/retrieval/test_assertions.py` → growing assertion library, new test cases come per fix ✅
- `dev/feature_debug/` → investigation module for documented problem ✅
- `verify_fix_works.py` → one-shot, does NOT belong in `dev/`. Worktree or `/tmp/` ❌

**Consequence:** when forensics or one-shot assertion is needed, you build the script in the worktree (not staged on merge — explicitly do not stage) or under `/tmp/`. When a one-shot assertion becomes a permanent regression guard, the test case is folded into an EXISTING `test_*.py` in `dev/` — no new file per fix.

## Structure

```
dev/
├── <area>/                        # One dir per area — SAME name as process-docs/<area>/
│   ├── 01_<report_script>.py      # produces a report → numbered
│   ├── <helper>.py                # produces no report → no number
│   ├── md/                        # report outputs (.md), file prefixed with the script number
│   │   └── 01_<label>.md
│   ├── csv/                       # report outputs (.csv)
│   └── png/                       # report outputs (.png)
└── cleanup/                       # Utility scripts (no area mapping)
```

## Naming Convention

**Only report scripts are numbered.** A script that PRODUCES a report gets a number (`01_`, `02_`, …). A script that produces no report carries no number. The number is not a dependency or position order — it marks "this one emits a report".

**The report carries the script's number.** A report-script `01_test.py` writes `md/01_testresults.md` (or `.csv` / `.png`) — same number prefix, so every report is traceable to the script that made it.

**Report outputs live in type-folders** inside the area dir: `md/`, `csv/`, `png/`. All three are possible, none is mandatory — a script writes to whichever type(s) it emits.

**Example:**
```
dev/retrieval/
├── 01_retrieval_eval.py        # emits a report → numbered
├── retriever.py                # no report → no number
├── md/
│   └── 01_retrieval_eval_baseline.md
└── csv/
    └── 01_retrieval_eval_sweep.csv
```

## Rules

1. **Area grouping** — top-level dev/ dirs correspond to areas, each named like its `process-docs/<area>/` folder (e.g., `indexing/`, `retrieval/`)
2. **Number only report scripts** — a script that produces a report is numbered (`01_`, `02_`, …); a script that produces no report is not.
3. **Report carries the script number** — output goes to `md/`/`csv/`/`png/`, and the file is prefixed with the producing script's number (`01_test.py` → `md/01_testresults.md`).
4. **Dev is self-contained** — dev code does NOT import from `src/`. Dev mirrors prod interfaces but is independent. When a dev implementation is proven, it gets migrated to `src/` (lean, without report output).
5. **cleanup/** — utility scripts without area mapping (data cleanup, migration). No area grouping needed.
6. **Reports never go to console** — a report-producing script writes to `md/`/`csv/`/`png/`, never dumps results to the terminal. Console is limited to the output file path.
7. **Python execution** — ALL Python commands MUST use `./venv/bin/python` (not `python` or `python3`). The system Python does not have project dependencies installed.
