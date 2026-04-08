# Monitor_CC Standards

**Keyboard + Mouse:** UI-Steuerung per Tastatur (Digits 1-9) und Maus (SGR mode). Expand/Collapse, Hover-Highlight, Scroll in Subagent- und Workers-Pane. Mouse via SGR protocol (`\033[?1003h` + `\033[?1006h` — Mode 1003 = Any Event Tracking incl. motion), verifiziert gegen tmux source (`input-keys.c:755-822`). tmux `mouse on` (Scroll) und App-Mouse-Mode koexistieren ohne Konflikt.

**Unbuffered stdin reads (CRITICAL):** All stdin reads in `click_handler.py` use `os.read(fd, 1)`, NOT `sys.stdin.read(1)`. Python's `sys.stdin.read(1)` reads a large chunk (4096+ bytes) into an internal buffer. Subsequent `select.select()` checks the OS fd (empty) and misses buffered data. This causes partial escape sequences to leak as digit keypresses → spurious toggles. `os.read(fd, 1)` reads exactly 1 byte from the OS fd, keeping `select` and reads in sync.

## Color Palette

Single source of truth: `src/constants.py`. ALL color constants defined there, nowhere else.

- 256-color ANSI codes (e.g. `\033[38;5;35m`) for semantic colors
- RESET defined once
- Modules import colors from constants.py, NEVER define local color constants

**Prohibited:**
- Color definitions in any file other than constants.py
- Redefining color names with different ANSI codes
- Using raw ANSI escape codes inline — always use named constants from constants.py

## Configuration Values

ALL tunable values (intervals, thresholds, limits) live in `src/constants.py`.

**Currently centralized:**
- POLL_INTERVAL, INPUT_POLL_INTERVAL
- LONG_OUTPUT_THRESHOLD, EXPANDED_MAX_LINES
- TMUX_HISTORY_LIMIT
- Tool names, mode strings, patterns, message type sets

**Rule:** When adding a new numeric threshold or interval — constants.py, not the module.

## Timestamps (UTC vs Local)

`hook_outputs.jsonl` stores timestamps in **UTC** (`datetime.utcnow().isoformat() + "Z"`). The monitor displays timestamps in **local time** (via `format_timestamp` in utils.py). When comparing timestamps between hook log entries and user-visible display output: always account for the UTC offset (CET = UTC+2, CEST = UTC+2).

## Module-Level State

Monitor uses module-level mutable state for polling loop shared data. This is accepted for the streaming architecture.

**Rules:**
- New global variables ONLY in INFRASTRUCTURE section
- Every new global MUST be documented in the module's DOCS.md entry
- Prefer extending existing state dicts over adding new globals
- No unbounded growth: every cache/buffer that grows per-event MUST have a cleanup path or TTL comment

## Import Convention

Colors and config values: import from `src/constants.py` directly.
Utility functions (format_timestamp): import from `src/utils.py`.

**Prohibited:**
- Importing colors from utils.py (legacy re-export only, not for new code)
- Importing colors from formatter.py or any other module

## tmux Format Variables

BEFORE using tmux variables (`#{...}`) in code: verify with `tmux display-message -p '#{var}'` or `tmux list-panes -F '#{var}'` that the variable exists and returns data. Not all variables exist in all tmux versions or contexts.

**Format expansion contexts:**
- `run-shell`: YES — expands `#{...}` before passing to shell
- `display-message -p`, `if-shell -F`: YES
- `-t` target arguments (`respawn-pane -t`, `send-keys -t`): NO — passed literally

**Known variable issues:**
- `#{pane_activity}`: does NOT exist. Use `#{window_activity}` via `list-panes -F`.
- Concrete failure (2026-03-27): `#{session_name}` in `respawn-pane -t` chain — tmux passed it literally. Fix: `run-shell` wrapper.
- Concrete failure (2026-03-27): `#{pane_activity}` empty. `#{window_activity}` works.

## Proxy Log Investigation

When searching for a specific message format in proxy logs (e.g., rejection messages, error patterns):
- **Trigger it yourself first.** Running `sleep 10` + ESC is 10x faster than grepping through 800MB JSONL with wrong patterns.
- Only grep logs when you need historical data or pattern frequency analysis.
- The monitor's Proxy Pane shows live requests — use it to see the current format.

Concrete failure (2026-04-09): 3 failed grep attempts searching for ESC-rejection message format in proxy logs. User triggered it themselves in 5 seconds.

## Pane Module Architecture

Each pane is its own module under `src/`. Pattern:
- `*_pane.py` contains: module-level state, `run_*_loop()`, helpers, formatting
- `monitor.py` is the core orchestrator (~460 lines), imports pane loops lazily
- `formatter.py` has shared formatting only (~230 lines)
- Cross-module state: pane modules access monitor.py globals via `from . import monitor as _monitor`

## Reference Patterns for Interactive Panes

When building a new interactive pane (scroll + expand/collapse + click), use the **Tokens Pane** (`run_tokens_loop()` in monitor.py) as reference. It has working:
- Virtual scroll via `scroll_offset` (mouse wheel button 64/65)
- Click to expand/collapse via `line_map`
- Hover highlight via mouse motion (button >= 32)
- `enable_mouse()` with full motion tracking (`\033[?1003h`)
- Screen refresh only when `output != last_output`

**Do NOT use the Rules Pane** as reference for dynamic content. Rules-Pane uses the same architecture but works because content rarely changes. For panes with frequent updates (Hooks, Workers), the full-screen-refresh causes flicker.

**Key constraint:** `enable_mouse()` captures ALL mouse events (including wheel) from tmux. tmux native scrollback (Ctrl+B [) does NOT work when mouse mode is active. Scrolling must be handled by the app via `scroll_offset`.

Concrete failure (2026-04-04): Hooks-Pane went through 5 iterations — Rules-Pane pattern, append-only, click-only mouse mode, keyboard-only, back to tokens-pane pattern. Should have studied Tokens-Pane first in PLAN phase.

## Proxy Edit Safety

mitmproxy **hot-reloads** addon scripts when the file changes on disk. This resets `ProxyAddon` state (`prev_messages_by_model`) → BP3 can't find unchanged prefix → cache invalidation.

**Live-Copy protection (implemented):** `claude_proxy_start.sh` copies `proxy_addon.py` to `.proxy_addon_live.py` and loads the copy. Git merges on the original don't trigger hot-reload. Worker proxies also use live-copy (`.proxy_addon_worker_{name}.py`) since iterative-dev commit b8930f3.

**Rules:**
- NEVER edit `proxy_addon.py` directly during a live session if the proxy was started WITHOUT live-copy (old proxy instances)
- When multiple proxy changes are needed: batch them in one worker, merge once = one potential reload
- Hot-reload cannot be disabled in mitmproxy (hardcoded `reload=True` in script.py)

Concrete failure (2026-04-08): Worker merged proxy_addon.py changes → mitmproxy hot-reloaded → state reset → 145k CC cache rebuild. Three separate merges in one session = three rebuilds.
Concrete failure (2026-04-09): Worker proxy (spawned before live-copy fix) loaded proxy_addon.py directly → git merge triggered hot-reload → 41k CC rebuild. Fixed: worker proxies now use live-copy.

## Anti-Patterns

- "Code erst aktiv bei Neustart" — restarting the monitor IS your job, not an excuse to skip verification
- Fixing without reproducing first — you don't know if your fix works if you never saw the bug
- Editing production code directly — use worktrees for isolation
- `ALL_DELIVERABLES_COMPLETE` without screenshot — NEVER. Test first, promise second.
- Asking user to press keys to test — if you have a screenshot tool, USE IT
