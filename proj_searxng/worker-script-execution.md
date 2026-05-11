# Worker Script Execution — No Sleep, No Background Polling

Workers MUST run dev scripts (smokes, probes, evals, benchmarks) in the foreground with an explicit Bash `timeout` parameter. The sleep+poll orchestration pattern that Opus uses for managing alive workers is NOT a tool for workers themselves — workers are leaf nodes, they execute and report.

## Rule

When running any `./venv/bin/python dev/...` script:

1. **Foreground only.** Single Bash call, `run_in_background=false` (default).
2. **Explicit timeout** sized to the script's actual runtime. Bash tool max is `600000` ms (10 minutes). Smokes that hit all 8 engines × 30 queries take ~7-8 min; pass `timeout=600000` to be safe.
3. **Redirect noisy output** to a file (`> /tmp/<name>.md 2>&1`), then `tail -50` or `grep` for the signal. Do NOT pipe verbose smoke output directly into context.

```bash
# RIGHT — foreground, generous timeout, file-redirected
./venv/bin/python dev/search_pipeline/01_google_smoke.py > /tmp/google_smoke.md 2>&1
# (Bash tool: timeout=600000)
tail -30 /tmp/google_smoke.md
```

## Forbidden

```bash
# WRONG — sleep+poll pattern (Opus orchestration leaking into worker)
Bash(command="./venv/bin/python dev/.../smoke.py > /tmp/out.md 2>&1", run_in_background=true)
Bash(command="sleep 120 && echo done", run_in_background=true)
# wait for completion
Bash(command="tail /tmp/out.md")
```

Any `sleep N` (foreground or background) in a worker session is a rule violation. The polling pattern produces zero useful output during the wait, costs extra tool-use exchanges, and hides the script's actual completion signal behind a separate poll cycle.

## Why workers ≠ Opus on this

- **Opus** uses background `sleep N && echo done` timers to poll WORKER status — that's orchestration of running tmux sessions, where the polled state lives outside Opus's process.
- **Workers** running scripts are the script's process boundary. There's nothing to "poll" — the script returns when done. The Bash tool's `timeout` parameter is the correct mechanism for "wait up to N seconds for this command to complete".

If a script genuinely takes longer than 10 minutes, that is a signal to **scope it down** (smaller `--max-queries`, smaller test pool, fewer engines), NOT to background-poll. The worker session is finite context — long polls burn budget for no progress.

## Concrete failure

Session 2026-05-07, worker `jqn-foundation` used 8+ background sleep timers (60s, 120s, 180s mixed) to wait on `01_google_smoke.py` and `11_pipeline_smoke.py` runs. The runs were verification steps for the Google selector fix and free-word hook. The correct call was a single `Bash(command="./venv/bin/python dev/search_pipeline/01_google_smoke.py > /tmp/out.md 2>&1", timeout=600000)` followed by reading `/tmp/out.md`. Instead the worker burned ~10 polling exchanges and the user surfaced the pattern as an obvious anti-pattern. Adding this rule because the failure mode is recurring across worker sessions in this project (long-running smokes are the norm here).
