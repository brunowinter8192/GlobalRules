# Dev Flow: Bug Fix & Feature Verification

## Bug Fix Flow

1. **Reproduce** — Start monitor, trigger the bug, screenshot BEFORE fixing
   - `python3 workflow.py --project <project>` (or use running session)
   - Trigger the condition that causes the bug
   - `./venv/bin/python dev/display/screenshot_panes.py` → Read PNG → confirm bug is visible
   - If bug not reproducible → STOP, inform user

2. **Isolate** — Use a worker in worktree for the fix, not direct edits on main
   - Production code changes go through workers
   - Exception: <10 LOC single-file changes can be direct

3. **Fix** — Implement in worktree, merge back

4. **Verify** — Restart monitor with fixed code, reproduce the scenario, screenshot
   - Kill old monitor session: `tmux kill-session -t <session>`
   - Start fresh: `python3 workflow.py --project <project>`
   - Trigger the SAME condition as step 1
   - `./venv/bin/python dev/display/screenshot_panes.py` → Read PNG
   - Compare: bug gone, no regressions in other panes

5. **Pass/Fail** — If any pane shows wrong content → investigate before committing

## Feature Flow

1. **Implement** in worktree via worker
2. **Start monitor** fresh with new code
3. **Trigger** the behavior that exercises the new feature
4. **Screenshot** → Read PNG → confirm feature works as expected
5. **Pass/Fail**

## Screenshot Tool

```bash
./venv/bin/python dev/display/screenshot_panes.py
./venv/bin/python dev/display/screenshot_panes.py --session monitor_cc_global
```

Output: `/tmp/monitor_cc_screenshot.png` — Read with Read tool for visual inspection.

## Window/Pane Layout Reference

4 tmux Windows (switch with Ctrl-b 0/1/2/3):

- **Window 0 "main":** Pane 0 = Main monitoring output, Pane 1 = Token profiling + session browser
- **Window 1 "rules":** Pane 0 = Rules display, Pane 1 = Hooks
- **Window 2 "workers":** Pane 0 = Workers display
- **Window 3 "debug":** Pane 0 = Warnings, Pane 1 = Subagent list

## When To Use

- Any change to `src/monitor.py`, `src/formatter.py`, `src/ui_mode.py`, `src/subagent_ui.py`
- Any change to tmux layout (`src/tmux_launcher.py`)
- Any new display feature or bug fix
- After worker merge that touches display code

## Non-Interactive Monitor Start (for Claude)

Claude can start the monitor without a real terminal:

```python
import subprocess, os
from src.tmux_launcher import launch_split_screen

original_run = subprocess.run
def patched_run(cmd, **kwargs):
    if isinstance(cmd, list) and 'attach-session' in cmd:
        return subprocess.CompletedProcess(cmd, 0)
    return original_run(cmd, **kwargs)

subprocess.run = patched_run
launch_split_screen('/path/to/project', False, os.path.abspath('workflow.py'))
```

Then verify with:
- `tmux list-windows -t <session>` — check 4 windows exist
- `tmux list-panes -t <session>:0` — check 2 panes (main+tokens)
- `tmux list-panes -t <session>:1` — check 2 panes (rules+hooks)
- `tmux list-panes -t <session>:2` — check 1 pane (workers)
- `tmux list-panes -t <session>:3` — check 2 panes (warnings+subagents)
- `tmux display-message -t <session>:<W>.<P> -p '#{pane_start_command}'` — verify correct mode per pane
- `tmux capture-pane -t <session>:<W>.<P> -p -S -20` — check content with scrollback
- `./venv/bin/python dev/display/screenshot_panes.py --session <session>` — visual verification

## Auto-Loop Exit Verification (MANDATORY)

Before outputting `<promise>ALL_DELIVERABLES_COMPLETE</promise>`:
1. Start monitor via non-interactive pattern above
2. Wait 3s for panes to populate
3. Screenshot → Read PNG → verify all panes render correctly
4. Only THEN output the promise

Concrete failure (2026-03-23): First `ALL_DELIVERABLES_COMPLETE` without any test. tmux layout bug (wrong split order) discovered only after user called it out.

## Screenshot-First Debugging (MANDATORY)

When debugging ANY tmux/display issue:
1. **Screenshot BEFORE** any fix attempt — establish visual baseline
2. Apply fix
3. **Screenshot AFTER** — compare visually
4. If no visible change after 2 attempts → STOP, research external sources (tmux docs, GitHub, source code)

NEVER ask the user "did it work?" — use the screenshot tool. NEVER rely on PID changes or return codes alone — visual verification is the only proof for display issues.

Concrete failure (2026-03-26): 5 attempts to fix Ctrl+R respawn without taking a single screenshot. Only after user pointed out the screenshot tool existed was it used — and it immediately showed respawn was working (PIDs changed, panes reset). The actual problem was UX (no visible feedback), not technical failure.

## Unit-Tests sind keine Live-Verification

Wenn ein Worker eine Pane oder ein Display-Feature gebaut hat und alle Unit-Tests grün sind, heißt das nicht dass das Feature live funktioniert. Unit-Tests laufen gegen synthetische Daten — ein ausgedachtes `line_map`, ein fester Pane-Width, ein konstruierter Scroll-Offset — und können drei reale Drift-Ursachen grundsätzlich nicht fangen.

Der erste ist Header-Wrap: ein langer Header in einer schmalen Pane wrappt visuell über mehrere Terminal-Zeilen, der `phys_row`-Counter stimmt dann nicht mehr mit der tatsächlichen Terminal-Position überein. Der zweite ist Mouse-Event-Routing: wenn der Mouse-Mode nicht in der richtigen Pane enabled ist oder ein anderer Prozess den Event abfängt, kommt beim Pane-Code gar nichts an — im Unit-Test aber schon, weil der Test den Event direkt als Input reingibt. Der dritte ist Content-Overflow: lange Zeilen die visuell wrappen ohne dass der Code das mitzählt lassen den `line_map`-Index desyncen, was im Test mit kurzen synthetischen Zeilen nie auftritt.

Workflow nach dem Merge einer Pane- oder Display-Änderung ist deshalb: Monitor-Session killen und neu starten (`tmux kill-session` und `python3 workflow.py --project ...`), das ursprüngliche Failure-Szenario reproduzieren, einen Screenshot via `dev/display/screenshot_panes.py` machen und visuell vergleichen. Unit-Tests sind notwendig aber nicht ausreichend.

Wenn du die Live-Verify in der aktuellen Session nicht ausführen kannst (kein Monitor auf, keine passenden Daten zum Triggern), schließ den zugehörigen Bead nicht. Markier ihn als „code-verified, live pending" und vermerk in einem STAND-Kommentar was in der nächsten Session konkret zu testen ist. Ein Display-Fix-Bead wird nicht auf Unit-Test-Basis geschlossen.

Failure 2026-04-21: Commit cf901cb hat pane-level scroll für die Workers-Pane eingeführt, 40 von 40 Unit-Tests waren grün. Der Live-Test durch den User 30 Minuten später zeigte dass Scrollen nicht funktionierte — das Wheel-Event erreichte die Pane gar nicht. Bead j69 musste wieder geöffnet werden. Ein Screenshot vor dem Close hätte das in 30 Sekunden gezeigt.

Failure 2026-04-23: Bead 2z6 monitor-align worker hat für `_aggregate_req_buckets` inline smoke-asserts gefahren — classify_tags LEAK/SUS verified, counter-delta INERT verified, IDX case verified, alle grün. Kommit 8cc77a8 merged. Nach Live-Neustart: proxy pane crasht sofort bei expand. Ursache: `render_turn.py` Zeile 129 nutzt `DIM`, Import-Statement listet nur `SOFT_RESET, RED, WHITE, YELLOW` — NameError. Die Smoke-Asserts haben das nicht gefangen weil sie die Funktion isoliert aufrufen, nicht das Modul in der echten Laufzeit-Umgebung importieren. Ein `python3 -c "from src.proxy_display.render_turn import render_turn_expanded"` hätte das in 0.5 Sekunden gefangen. Ein zweiter Bug in derselben Arbeit — dass EFFECTIVE-Bucket bei counter-delta ALLE chunks zeigt statt nur neuer — ist datenabhängig (REQ-Historie) und wäre durch Live-Run auf der aktuellen Session-JSONL sofort aufgefallen; Smoke-Asserts mit synthetischen 2-Entry-Dicts haben das nicht gesehen.

## Pre-Commit Live Checks (MANDATORY für Worker an display/proxy-display/Rendering)

Statt „unit assertions passed" als Completion-Signal braucht der Worker drei konkrete Checks vor dem Commit. Jeder ist in unter 60 Sekunden gemacht und fängt genau die Fehler-Klassen die Smoke-Asserts nicht sehen.

### Check 1 — Live Import (fängt NameError / Missing-Import in 0.5s)

Für jedes geänderte Modul:

```bash
./venv/bin/python3 -c "from src.proxy_display.render_turn import render_turn_expanded"
./venv/bin/python3 -c "from src.proxy_display.render_messages import render_messages, _aggregate_req_buckets"
./venv/bin/python3 -c "from src.proxy.strip_vocab import classify_req, classify_tags, attribute_chunk"
```

Wenn ein Import wegen fehlender Konstante (DIM), falschem Symbol, oder Typo im relativen Pfad scheitert, knallt das hier — nicht erst live nach dem Merge mit totem Proxy-Pane.

### Check 2 — Ad-hoc-Script gegen echte JSONL (fängt Daten-Pattern-Bugs die Smoke-Asserts nicht sehen)

Wenn die Änderung Rendering oder Klassifikation von Proxy-Log-Entries betrifft, schreib ein kurzes `/tmp/verify_<feature>.py` das:
1. Echte JSONL einliest (aktuell laufende Session aus `src/logs/api_requests_opus_monitor_cc_*.jsonl`, auto-newest)
2. Das geänderte Modul importiert (Live-Import-Check inklusive)
3. Die geänderte Funktion auf 20+ echten Entries ausführt, Output sammelt
4. Einen konkreten Anchor-Fall aus dem Failure-Kontext assertet

Template das Opus im Worker-Prompt mitliefert wenn das Feature datenabhängig ist:

```python
import sys, os, json, traceback
from pathlib import Path

sys.path.insert(0, os.path.abspath('.'))
from src.proxy_display.parser import _parse_log_file
from src.proxy_display.render_messages import _aggregate_req_buckets

log = sorted(Path('src/logs').glob('api_requests_opus_monitor_cc_*.jsonl'),
             key=lambda p: p.stat().st_mtime, reverse=True)[0]
entries, _ = _parse_log_file(log, 0)
entries = [e for e in entries if (e.get('model') or '').startswith('claude-opus-')]
print(f'loaded {len(entries)} opus entries from {log.name}')

for i, entry in enumerate(entries):
    prev = entries[i-1] if i > 0 else None
    try:
        buckets = _aggregate_req_buckets(entry, prev)
        # anchor assertion specific to current change:
        # e.g. assert REQ#109 NAG nicht auf denselben msgs wie REQ#99
    except Exception:
        print(f'REQ#{i+1} FAIL')
        traceback.print_exc()
        break
```

Laufzeit: 2-5 Sekunden. Fängt genau die Bugs die Smoke-Asserts mit gemockten 2-Entry-Arrays nicht sehen.

### Check 2a — Top-Level Render-Funktion aufrufen (fängt Variable-Scope-Bugs die Helper-Calls übersehen)

Check 2's "geänderte Funktion ausführen" kann missverstanden werden. Wenn die Änderung in einem Helper liegt (z.B. `_aggregate_req_buckets`, `classify_tags`, `_is_standalone_entry`), reicht ein isolierter Helper-Aufruf nicht. NameErrors und Variable-Scope-Bugs im umgebenden Render-Code tauchen erst auf wenn die top-level Render-Funktion ausgeführt wird. Der Import-Check fängt AST-Fehler. Der Helper-Call fängt Helper-Logik-Fehler. Den Rahmen drum rum fängt nur ein echter End-to-End-Call.

Für jede Änderung an `src/proxy_display/`: dein Ad-hoc-Script ruft zusätzlich zum Helper-Call die top-level Funktion der betreffenden Pipeline auf, mindestens einmal, mit einer REQ deren Expand-State aktiv ist:

```python
from src.proxy_display.render_turn import render_turn_expanded

# Group mit realen Entries und mindestens einer REQ die expanded ist
group = {'entry_pairs': [(i, entries[i]) for i in range(min(7, len(entries)))]}
expand_states = {('req', 6): True}  # REQ an Index 6 expanded
lines, keys, _, _ = render_turn_expanded(
    group, entries, expand_states, pane_width=180,
    prev_entry_for_delta=None, opus_req_num=0, sub_req_num=0,
)
print(f'render_turn_expanded → {len(lines)} lines, no exception')
```

Kein pane_width-Kunststück nötig, keine Screenshots — nur der Call. Wenn irgendeine Variable im expanded-Branch undefined ist, crasht der Aufruf hier, nicht erst in der Live-TUI nach Merge. Laufzeit: unter einer Sekunde.

Failure 2026-04-23 (zweiter Anlauf): Worker ref-alignment commit 4928b11. Rename `_delta_ref` → `_section_ref` war in vier Call-Sites nötig, Worker hat nur drei aktualisiert. `m_lines = render_messages(entry, _delta_ref, entries, ...)` blieb unverändert. Live-Import passed (AST kompiliert sauber, der Fehler ist im Funktionskörper). Check 2 passed (der ad-hoc Script hat `_is_standalone_entry` isoliert verifiziert, nie `render_turn_expanded`). Ein einzelner `render_turn_expanded(...)` Call mit `expand_states={('req', 6): True}` hätte die NameError sofort geliefert. Nach Opus-Korrektur hat der Worker genau diesen Call als Anchor C dem Script hinzugefügt — beim ersten Run gefangen.

### Check 2b — Interactive Features brauchen E2E-Simulation, nicht isolierte Helpers

Features that depend on keyboard/mouse input (y-hotkey, click-expand, drag-select, hover-tracking) cannot be validated by calling helper functions in isolation against mocked data. The bug surface includes the WHOLE input → resolve → serialize → output pipeline. If the verify-script only calls the last function of the pipeline, it misses bugs in the earlier stages.

When writing `/tmp/verify_<feature>.py` for an interactive feature, the script MUST:

1. Load the real data source (session JSONL, proxy log) the feature will consume.
2. Simulate the user's exact input-path on that real data:
   - Click-to-expand: pick a specific row, call the click-handler logic with that row → verify expand state + re-render
   - Hover+y: set `hover_row = <known_row>`, call the full resolve_parent_key → serialize → output chain → verify output content
   - Serialize: NEVER call serializer directly with hand-crafted key on synthetic data. Always go through the key-resolution that the user's input would produce.
3. Include MULTIPLE REQ positions (early, middle, late) — bugs that only appear on middle-session REQs (where cumulative state grows) are easy to miss with only early-session tests.
4. Output assertion includes a concrete char-count or output-head for a MID-SESSION position, not just the first entry. If the char-count or head looks "too large" for a single REQ's content, flag as scope bug before merge.

Concrete failure (2026-04-23): main-clipboard worker wrote `/tmp/verify_main_clipboard.py` for Main-pane y-hotkey. Script called `serialize_main_event(event_idx)` directly on a mocked `main_event_buffer` with synthetic events. Passed all assertions. Never simulated `resolve_parent_key` → `serialize_main_event` → `copy_to_clipboard` on real session data. The cumulative-history serializer bug in `_serialize_proxy` (shared pattern) escaped detection because the verify-script's scope was helper-only. Bug surfaced only when user manually clicked + pressed y in live Monitor — 700kB clipboard dump. A verify-script replaying `(real_jsonl_entries, hover_row=<REQ20_header>)` through the full pipeline would have shown the bug in 2 seconds.

### Check 3 — Screenshot post-Merge als Gate (Opus-Verantwortung)

Worker commited → Opus reviewt → Opus merged auf dev → Opus macht Monitor-Restart → Opus screenshotet den relevanten Pane-State → Opus validiert visuell gegen die Anchor-Fälle.

NICHT: Worker's „unit assertions passed" als Abschluss-Signal akzeptieren. NICHT: Bead schließen auf Basis der Self-Reports. Bead bleibt offen bis Opus das Screenshot gesehen hat und der User zustimmt.

Der Restart-Schritt ist Opus's Aufgabe, nicht optional:
```bash
tmux kill-session -t monitor_cc_<hash>
./venv/bin/python3 <restart_script.py>  # non-interactive launch_split_screen pattern
sleep 10
./venv/bin/python3 dev/display/screenshot_panes.py --session monitor_cc_<new>
# Read /tmp/monitor_cc_screenshot.png → visual check
```

### Worker-Prompt-Template

Für zukünftige Display-Worker: ersetze in der Completion-Checklist „unit assertions passed" durch:

```
- [ ] Live-Import Check: `python3 -c "from <module> import <new_symbols>"` OK für jedes geänderte Modul
- [ ] Ad-hoc JSONL-Script Check (wenn daten-abhängig): `/tmp/verify_<feature>.py` gegen aktuelle Session läuft ohne Exception und echte Anchor-Assertion passt
- [ ] Top-Level Render-Call Check (für alle src/proxy_display/ Changes): das ad-hoc Script ruft zusätzlich `render_turn_expanded(group, entries, expand_states={('req', N): True}, ...)` oder die äquivalente top-level Render-Funktion auf — Helper-Calls alleine fangen Variable-Scope-Bugs im Render-Frame nicht
- [ ] Committed auf worktree branch (NO MERGE, NO SCREENSHOT CLAIMS — das macht Opus post-Merge)
```

Opus's verantwortlicher Schluss-Schritt bleibt außerhalb des Workers.

## Measurement Discipline (RAM / Latency / Performance)

When the user's goal is "reduce X" / "make Y faster" / "lower Z", Phase 1 Investigation MUST include actual measurement of WHERE X comes from, not just code-reading. Code-reading shows the SHAPE of the system; measurement shows the COSTS. The two are different. Without measurement, "this list grows unbounded" looks like the dominant cost when the parse-peak ahead of the list is actually dominant.

### Measurement Before Optimization

For RAM tasks, mandatory steps before dispatching a worker:

1. `ps`-snapshot at the dimension being optimized (per process, per pane, per subsystem) — concrete RSS baseline.
2. ONE of these instrumentation paths, executed BEFORE worker-dispatch:
   - `tracemalloc` snapshot at suspected hot points → top allocators by file:line
   - `gc.get_objects()` + `collections.Counter` over `type(o)` → class-distribution of live objects
   - `gc.get_referrers(obj)`-walk from a sampled large object → finds the holder pinning memory
   - `memory_profiler` with `@profile` decorator on a suspected hot function → line-level allocation increment
3. Identify the dominant allocation source by EVIDENCE, not by reading "list.append() exists in this loop".
4. State the measurement explicitly in the PLAN: "Measured peak RAM is dominated by X at file:line, accounting for Y MB of the Z MB total."

For Latency tasks:

1. Wrap suspect input/render boundary with `time.monotonic_ns()` markers.
2. Drive ≥100 events through the system (synthetic stdin, fake mouse, scripted clicks).
3. Compute median, p95, max latency.
4. Identify dominant component (poll wait, parse cycle, format pass, ANSI write, terminal flush).
5. State in PLAN: "Measured input latency median 25ms, p95 48ms, dominated by `time.sleep(50ms)` floor."

If measurement contradicts code-reading hypothesis → pivot worker scope. If measurement confirms → dispatch with confidence.

Anti-pattern: Phase 1 consisting ONLY of code-reading + grep, followed by worker-dispatch with rationale "the list grows so we cap it" / "the sleep is fixed so we replace with select" — without ever measuring what dominant cost actually is. This produces regressions.

Concrete failure (2026-04-25): RAM-Performance Session. Phase 1 read pane-modules and identified module-level lists growing unbounded (tool_errors, _waste_all_events). Cap-them dispatched. Live-verify after merge: waste pane RSS went from 1167 MB to 3067 MB — net regression. Root cause discovered post-mortem: dominant memory cost is `f.read()` peak in `_parse_log_file` (358 MB → 2-3 GB Python heap) plus O(N²) cumulative messages per proxy entry. Neither visible from code-reading alone; both obvious from a single `tracemalloc` snapshot. Cost: 1.5 worker cycles for regression + revert + streaming-fix attempt. The 5-minute measurement step would have prevented all of it.

### Measurement Scope — Instrument ALL Candidates

When the goal is "find which of N candidates is dominant" (RAM, latency, cost, error rate), instrument ALL N candidates from the start. Do NOT pre-filter to "the suspected one" based on historical data, prior assumptions, or rules-of-thumb. Pre-filtering produces single-data-point measurements that cannot be compared against the population they came from — you don't know if your suspect is actually dominant or just notably-high-but-not-the-largest.

The cost of broader instrumentation is usually one extra worker dispatch (refactoring inline boilerplate into a shared helper, applying to all candidates). The cost of narrow instrumentation is an additional measurement round AFTER user pushback, plus all the planning that depended on partial data.

At the start of a measurement task, ask: "what's the population I'd want to compare against?" Instrument the FULL population, not just the named suspect.

Concrete failure (2026-04-28): RAM-audit Phase 1 instrumented only the waste-pane based on a 2026-04-25 STAND comment that said "waste was historically heaviest". Live data later revealed warnings was 2.86 GB (second-largest after historical waste) — same proxy-log-parse pattern, same root cause. User pushback: "warum haben wir mit ram audit nur das waste pane einbezogen das macht doch keinen sinn? wir wissen ja garnicht was die anderen panes ziehen". Cost: additional worker round (q0t) refactoring waste-pane's inline RAM-handler into shared helper + applying to all 8 remaining panes. Original-scope full instrumentation would have saved ~30% of Phase 1 planning work.

## Dev-Script Output Discipline

### Commit Dev-Script Outputs Immediately

Monitor_CC's dev-scripts (`dev/tool_use_analysis/extract_*.py`, `dev/ram_audit/`, `decisions/OldThemes/*` parking files) regularly produce report MD files during a session. Pattern: run script → MD lands in `dev/<topic>/` → file is auto-staged but not committed.

When a worker is dispatched to merge a branch back into main, every uncommitted-but-staged file in main's working tree that overlaps the merge surface OR is a new file blocks the merge ("your local changes would be overwritten" or aborts mid-merge). Recovery dance: `git stash push -u → merge → git stash pop` per merge.

Rule: after running ANY dev-script that writes to `dev/<topic>/` or `decisions/`, commit the artifact immediately:

```bash
gc "chore: <script-name> session report — N entries / Y findings"
```

Don't accumulate artifacts unstaged-but-staged across the session. The artifacts are valuable analysis output — committing them costs nothing AND eliminates merge friction. Same for `decisions/OldThemes/<topic>.md` parking files when a bead is closed-as-parked.

Concrete failure (2026-04-28): 7 staged files (1 decision + 6 tool_use_analysis reports) accumulated across the session. Caused 4 stash-merge-pop cycles (e8ae197, 397e17a, cf073db, b8d4795). Each cycle: 3 separate git operations. ~12 extra git commands total + cognitive overhead.

### dump_all.sh — Per-PID Wait, Not Fixed Sleep

`dev/ram_audit/dump_all.sh` currently sends SIGUSR1 to all PIDs, then `sleep 1`, then lists fresh dumps. For panes NOT mid-deep-parse, the handler completes in milliseconds. For panes currently inside a JSON-decoder loop (warnings during full-rescan, proxy/metadata mid-large-file-read), Python's signal delivery waits for next byte-code-instruction boundary — can be hundreds of ms when stuck in C extension.

Rule: dump_all.sh polls per-PID for the dump file's existence with a max-timeout, not a fixed sleep:

```bash
for pid in $pids; do
  kill -USR1 $pid
done
deadline=$(($(date +%s) + 15))  # 15s max wait
while [ $(date +%s) -lt $deadline ]; do
  expected=$(ls /tmp/.monitor_cc_pid_* | wc -l)
  fresh=$(find dev/ram_audit/dumps -name "*.txt" -newermt "@$start" -type f | wc -l)
  [ "$fresh" -ge "$expected" ] && break
  sleep 0.5
done
```

Concrete failure (2026-04-28): first dump_all.sh run got 3/9 dumps. Second manual retrigger with sleep 10 got 7/9. Warnings only responded after a third targeted retrigger (kill -USR1 + sleep 5).

## Anti-Patterns

- "Code erst aktiv bei Neustart" — restarting the monitor IS your job, not an excuse to skip verification
- Fixing without reproducing first — you don't know if your fix works if you never saw the bug
- Editing production code directly — use worktrees for isolation
- `ALL_DELIVERABLES_COMPLETE` without screenshot — NEVER. Test first, promise second.
- Asking user to press keys to test — if you have a screenshot tool, USE IT
