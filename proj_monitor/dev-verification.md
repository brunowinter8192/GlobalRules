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

## Anti-Patterns

- "Code erst aktiv bei Neustart" — restarting the monitor IS your job, not an excuse to skip verification
- Fixing without reproducing first — you don't know if your fix works if you never saw the bug
- Editing production code directly — use worktrees for isolation
- `ALL_DELIVERABLES_COMPLETE` without screenshot — NEVER. Test first, promise second.
- Asking user to press keys to test — if you have a screenshot tool, USE IT
