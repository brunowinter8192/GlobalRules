# 2026-05-19 — Monitor_CC: menubar overhaul findings

## ~/.claude/shared-rules/proj_Monitor_CC/menubar.md (NEW project rule file)

**Problem:** launchd-spawned scripts in this project (menubar) erben NICHT user-shell PATH. tmux at `/opt/homebrew/bin/tmux` ist nicht in launchd's default `/usr/bin:/bin:/usr/sbin:/sbin` → silent failures in subprocess calls.
**How it should be:** plist for any launchd-spawned script muss `EnvironmentVariables/PATH` setzen.
**Rule:** Jede neue plist in diesem Projekt MUSS einen `EnvironmentVariables`-Block mit PATH inkl. `/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin` enthalten. Falls plist Placeholder-Tokens nutzt (`<PROJECT_ROOT>`), MUSS ein `setup_*.py` Script existieren das substituiert + bootstrap macht. Manuelles Copy ohne sed-Substitution erzeugt invalid plist mit XML-Error.
**Concrete example:** Session 2026-05-19 — Worker-Display-Bug. discover sah 6 sessions im standalone-Python-Test, menubar-process sah nur 2. Root cause: tmux nicht im PATH des launchd-spawned process. Fix: PATH in plist gesetzt + 2nd-try-bootstrap-retry-pattern (Bootstrap fails mit I/O error oft beim ersten Try).

## ~/.claude/shared-rules/proj_Monitor_CC/menubar.md (same file, second entry)

**Problem:** Restart-Button für rumps/NSApp-basierte Apps via `launchctl kickstart -k` killt den alten Process nicht (SIGTERM blocked by NSApp runloop). Resultat: doppelte Bar-Icons, Singleton-Bypass.
**How it should be:** Restart via `os.execv(sys.executable, [sys.executable] + sys.argv)` — in-place-Replace, gleicher PID, atomic.
**Rule:** Für jede rumps/NSApp/PyObjC-App in diesem Projekt: Restart-Pattern ist `os.execv`, NICHT `launchctl kickstart -k`. Singleton-Lock muss FD_CLOEXEC haben damit execv den Lock re-acquiren kann.
**Concrete example:** Session 2026-05-19. Restart-Button erzeugte Doppel-Bar-Icons über mehrere Stunden. Fix: os.execv + Singleton-FD_CLOEXEC.
