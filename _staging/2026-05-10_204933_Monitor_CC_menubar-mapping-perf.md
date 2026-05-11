# 2026-05-10 — Monitor_CC: Menubar Mapping + Perf Session

## ~/.claude/shared-rules/global/verify-before-execution.md → "Verify Inputs (Execution)" section, add new bullet

**macOS `pgrep -x` against pathful `comm`:** on macOS, `ps -p <pid> -o comm=` returns the FULL absolute path of the executable (e.g. `/Applications/Ghostty.app/Contents/MacOS/ghostty`), not the basename. `pgrep -x <basename>` against such processes silently returns no match — no error, just empty stdout. `pgrep -f <substring>` also unreliable for some bundled GUI apps.

Symptom: a worker probe reports `pgrep -x ghostty` "works" in isolation, then post-merge the same call returns nothing on dev. The discrepancy is environmental (some shell/subprocess contexts pgrep matches differently) and impossible to reproduce reliably.

Pattern that works deterministically on macOS:

```python
r = subprocess.run(['ps', '-A', '-o', 'pid=,command='], capture_output=True, text=True, timeout=2)
for line in r.stdout.splitlines():
    if '<unique-path-substring>' in line:
        pid = line.split(None, 1)[0].strip()
        if pid.isdigit():
            return pid
```

Rule: when looking up a macOS GUI app's PID, do NOT use `pgrep -x` or `pgrep -f` — parse `ps -A -o pid,command` directly and substring-match against a unique path fragment of the app bundle (e.g. `Ghostty.app/Contents/MacOS`). This removes the entire pgrep-behavior surface from the equation.

Failure 2026-05-10: menubar mapping worker dispatched with `pgrep -x ghostty` after Phase A "live test" reported it returning the right PID. Post-merge, same call returned None on dev → click-jump silently broken until ps-direct-parse fix landed.

## ~/.claude/shared-rules/worker/worker-rules.md → "Pre-Commit Live Checks" section, add new sub-section

**Verify-Script muss tatsächlich den im neuen Code geänderten Pfad ausführen.** Eine "verify_<feature>.py PASSED" Aussage ist NUR dann verlässlich wenn das Script genau die Funktion aufruft die durch deine neue Logik gegangen ist. Wenn dein Verify zufällig auf einem aufgewärmten Cache läuft (z.B. die zu prüfenden Daten sind bereits aus einer vorherigen Probe in einem Module-Level-Dict gelandet), dann ist PASSED nichts wert — die fehlerhafte neue Logik wurde nie betreten.

Konkret: bevor du "verify PASSED" reportest, beantworte für dich:

1. Welche neue Funktion löst dein Verify-Script aus? Datei + Zeile.
2. Welcher Code-Pfad ist NEU (von dir in dieser Phase B geändert)? Wenn das Verify-Script den nicht trifft, ist der PASSED-Claim nicht aussagekräftig.
3. Hat dein Script eine RESET-Phase die alle Module-Caches leert bevor es den ersten Call macht? Sonst läuft es gegen Stale-State.

Pattern für State-volle Module:

```python
# RESET state explicitly before timing/verifying
import importlib, src.menubar.discover as d
importlib.reload(d)  # clears module-level caches like _ghostty_tty_to_id
# ... THEN call the function under test
```

Failure 2026-05-10: menubar mapping worker reportet `verify_mapping.py PASS` nach pgrep-x-bug-introduction. Das Script lief korrekt — aber der Cache-Eintrag den es prüfte stammte aus einem früheren probe-Aufruf in derselben Python-Session, NICHT aus dem aktuell fehlerhaften Code. Erst Opus-side post-merge re-run mit frischem Python-Prozess hat den Bug aufgedeckt.
