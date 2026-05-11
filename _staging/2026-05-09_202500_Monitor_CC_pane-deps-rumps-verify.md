# 2026-05-09 — Monitor_CC: tmux-pane deps + rumps menu verify

## ~/.claude/shared-rules/proj_Monitor_CC/pane-deps.md → NEW FILE (or add to existing project rule)

### Tmux-Spawned Pane Modules — System-Python Compatibility

Modules invoked via `tmux new-window … python3 workflow.py --mode <X>` run under whatever `python3` is on PATH at the time of `tmux new-window`. In typical setups that's `/opt/homebrew/bin/python3` (system Python), NOT `/Users/.../Monitor_CC/venv/bin/python3`.

System Python on this machine has only stdlib + `pillow`. It does NOT have `httpx`, `rumps`, `tiktoken`, `mitmproxy`. Any pane module whose top-level imports include a venv-only dependency will:

1. fail with `ModuleNotFoundError` on launch
2. cause tmux to immediately close the window (default `remain-on-exit off`)
3. produce a SILENT bug — no Window appears, no error visible, no log entry

Concrete failure observed 2026-05-09 (gpu-pane Window 4): `import httpx` at module top of `src/gpu_pane/status.py` → Window 4 silently never appeared after `python3 workflow.py --project ...`. Fix was `import urllib.request` (stdlib) instead.

**Rule for new tmux-spawned pane modules:**

1. Top-level imports MUST be stdlib OR known-system-installed packages (`pillow`).
2. If a non-stdlib package is genuinely needed (rumps for menubar):
   - either move the import inside the `run()` entry point (no module-load failure, only call-time failure with a clear traceback)
   - OR change `*_cmd` in `src/tmux_launcher.py` to use the venv python explicitly: `f"{VENV_PY} {script_path} --mode <X> {project_arg}"` where `VENV_PY = os.path.join(os.path.dirname(script_path), "venv", "bin", "python3")` (with stdlib fallback if venv missing).
3. Verify before merging: run the exact `python3 workflow.py --mode <X>` command via `/opt/homebrew/bin/python3` (system) AND `./venv/bin/python3` (venv). Both must succeed for at least 2 seconds.

The menubar app is the existing exception — it requires `rumps` and the user explicitly launches it via `venv/bin/python3`. NOT launched by `tmux_launcher.launch_split_screen`. Documented in `src/menubar/README.md`.

---

## ~/.claude/shared-rules/opus/opus-workers-2.md → "Pre-Commit Live Checks" section

### rumps Menu API — Test ≥2 Tick Behavior

When a worker implements a rumps-based menubar app and uses `app.menu = items` (or `Menu.update()` semantics), the verify script MUST exercise the menu rebuild ACROSS ≥2 ticks against the same input, then assert that the menu item count is stable.

`app.menu = list` in rumps does NOT atomically replace — it iterates `update()` which calls `_choose_key` (a unique sentinel object per call) so `__setitem__` always inserts. Visual NSMenu accumulates entries across ticks even though the dict-level state stays unique-by-title. Single-tick verify scripts will NOT catch this.

**Concrete pattern in `/tmp/verify_<feature>.py`:**

```python
app = CCMenuBarApp()
sessions = [SessionInfo(...), SessionInfo(...), SessionInfo(...)]
_rebuild_menu(app, sessions)
n_after_first = len(list(app.menu))

_rebuild_menu(app, sessions)  # SAME sessions, second tick
n_after_second = len(list(app.menu))

assert n_after_first == n_after_second, f"menu accumulated: {n_after_first} → {n_after_second}"
```

Right replacement idiom in rumps: `app.menu.clear()` first, then `app.menu.add(item)` per item. Re-add `app._quit_button` explicitly after clear because rumps' auto-managed quit-button is also wiped by `clear()`.

### tmux-Spawned Mode — System-Python Smoke Test (Pre-Commit)

For any new `--mode <X>` route in `workflow.py` that runs as a tmux window (not just on-demand utilities like `--mode menubar`), the worker's pre-commit live checks MUST include:

```bash
/opt/homebrew/bin/python3 -c "from src.<package>.<entry> import <function>; print('ok')"
```

with the system Python explicitly. NOT just `./venv/bin/python3`. If this fails, the new mode will silently fail to spawn under tmux when `python3` resolves to system Python via PATH.

If the import fails on system Python: either add a stdlib alternative for the failing import, OR explicitly route via venv python in `tmux_launcher.py:_build_mode_commands`. Document the chosen path.
