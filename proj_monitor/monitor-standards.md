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

**Proxy JSONL is also UTC.** `src/logs/api_requests_*.jsonl` `.timestamp` fields carry the `Z` suffix. When correlating user-reported events ("at 22:00 local", "just now") with proxy log entries, convert explicitly and state the conversion. Do NOT assume your first interpretation matches the user's timezone.

Concrete failure (2026-04-15): Confused "22:15 local" with "22:15 UTC" in proxy log search. Burned 2 tool calls on wrong time ranges before realizing `20:14 UTC` entries = `22:14 local`.

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

## Pane Module Architecture

Each pane is its own module under `src/`. Pattern:
- `*_pane.py` contains: module-level state, `run_*_loop()`, helpers, formatting
- `monitor.py` is the core orchestrator (~460 lines), imports pane loops lazily
- `formatter.py` has shared formatting only (~230 lines)
- Cross-module state: pane modules access monitor.py globals via `from . import monitor as _monitor`

### Header + Body Pane Contract (MANDATORY)

When a pane renders `header + body` and body comes from a formatting function:

1. VERIFY the body function respects its `pane_height` / `pane_width` inputs as HARD limits
2. If body lines can exceed `pane_width` → visual wrap multiplies rows → body overflows and pushes the header off the top
3. Default-safe fix: ANSI cursor overdraw after body print:
   ```python
   print(output, end='', flush=True)
   print(f"\033[H{header}\033[K", end='', flush=True)
   ```
   This redraws the header at row 0 regardless of body size. `\033[K` clears EOL.
4. Never rely on `print(output)` with implicit newline — it adds one row that pushes content off the top.
5. Verification MUST use live data with long body content (not "no workers" / "no errors" empty case). Empty-body tests pass trivially and miss the wrap bug.

Concrete failure (2026-04-15): 3 iterations on worker-proxy + warnings header. Worker's first fix verified with empty body ("no workers"), passed. Real load had 2 live workers + 700+ tool errors, body overflow pushed header off. Only visible when testing with production-like data.

### Similar-Named State Variables — refactor or document the pair

When `*_pane.py` introduces a state variable whose name has the same stem as an existing variable (singular vs plural, `_offset` vs `_offsets`), it is a smell — either the two are synchronized (and the relationship should be documented in a comment), or one is dead, duplicated, or not actually being hit by the handler. During bug investigation in such panes FIRST CHECK: who writes which, who reads which. String-comparison of names alone is enough to surface the mismatch.

Rule: when adding a new state variable in `*_pane.py`, if a variable with the same stem already exists, EITHER extend the existing one OR replace it entirely. Side-by-side without an explicit comment documenting the relationship is a bug invitation.

Concrete failure (2026-04-22, worker_pane.py): commit cf901cb added `worker_scroll_offset` (int), but `worker_scroll_offsets` (dict) already existed in `worker_format.py:129`. Handler wrote int, `format_cache_tracker` read dict. No synchronization, no documentation, live-broken. Two-variable confusion surfaced only after a worker round and a live test — string-comparison at first investigation would have caught it instantly.

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

## Display Feature Scoping (MANDATORY)

Before dispatching a worker for ANY display/pane change:
1. **Read the display function** that renders the affected area (e.g., `format_proxy_block()` for proxy pane)
2. **Describe the current output** — write a concrete example of what is shown now
3. **Describe the desired output** — write a concrete example of what should be shown
4. **Identify the data gap** — does the rendering function have access to the data needed? If not, which data source needs enrichment?

Without this: the worker implements something that "works" but doesn't address the actual UX problem.

Concrete failure (2026-04-10): Worker-Dashboard dispatched without understanding that `format_proxy_block()` shows ALL messages when expanded instead of delta-only. The delta header (Δmsgs) was correct, but the expanded message list was a flat dump. 3 worker corrections + user complaints needed to discover this.

### Empirical UI Self-Check After First Merge

After the first merge of a UI-affecting feature, run an empirical check BEFORE dispatching follow-up workers. Capture a screenshot (`dev/display/screenshot_panes.py` or user-provided) on a data-point where the feature should be visible, read the PNG, compare against the user's stated placement/format/intent. Iterate only if the screenshot matches. Does not apply to non-UI changes.

Concrete failure (2026-04-22): tag-viz first merge (`0b01357`) put suspect-tag badge at BLOCK-header level. User had originally said "im req header inline" in the initial scope request. Misplacement caught only after the user inspected the live output and flagged it ("wir brauchen po und sr im req header inline und nicht in der einzelnen msg"), requiring a second worker_send to move the badge up. A 30-second screenshot + compare at end of first merge would have caught this before the second dispatch.

## Cache / Proxy Forensics

### Proxy Addon Live-Copy — rules.py changes are not hot-reloaded

The mitmproxy addon is NOT loaded at runtime from `src/proxy_addon.py`. `claude_proxy_start.sh` copies a static snapshot to `src/logs/.proxy_addon_live_<SESSION_ID>.py` at session start and runs mitmproxy with `-s <this-copy>`. The running proxy reads `rules.py`, `content_strip.py`, `strip_sr.py` etc. only ONCE at start. The live-copy is deleted at session end.

Consequence for fixes to `src/proxy/rules.py`, `src/proxy/content_strip.py`, `src/proxy/strip_sr.py`, `src/proxy/payload_helpers.py`, `src/proxy/inject_helpers.py`: the fix does NOT affect the running CC process, no matter how often Ctrl+R is pressed in the Monitor session. Ctrl+R respawns Monitor panes, not mitmproxy.

To bring a proxy fix live:
1. End the CC main session (proxy dies with the CC process tree)
2. Restart CC → `claude_proxy_start.sh` writes a fresh `.proxy_addon_live_*.py` based on current `proxy_addon.py` and imports
3. New API requests go through the fixed code. Old log entries are unchanged (written by pre-fix proxy) — not retroactively healable without a separate migration script that re-runs the rules

When user reports "I don't see the fix after Ctrl+R", that is the FIRST diagnosis direction before doubting code or display: the proxy is still running with the pre-fix snapshot.

`rules_config.py` has mtime-reloads for `proxy_rules.json` and `shared-rules/` files — that's a separate, independent pipeline and does hot-reload (it also triggers cache rebuilds, see "Cache Rebuild Investigation" below). The strip-code path itself is NOT hot-reloaded.

Concrete failure (2026-04-22): Strip-tracking fix (Bead 9mf) merged on dev, user pressed Ctrl+R, panes showed pre-fix state. Opus didn't proactively communicate that CC-session-restart was needed. The merge report should have flagged it.

### Monitor Pane Code IS hot-reloaded — opposite of proxy

The pane processes (proxy pane, tokens pane, workers pane, hooks pane, warnings pane, etc.) are independent Python processes spawned via tmux. When the user presses `Ctrl+R` in tmux, `respawn-pane -k` kills and re-launches the pane's `python3 workflow.py --mode <pane>` command. The newly spawned Python imports from current disk state — Monitor pane code changes ARE picked up by Ctrl+R, without Monitor-wide restart.

This is the OPPOSITE of the proxy addon (above). Test before making claims about whether a fix is live:
- Bug in `src/proxy/` (strip rules, addon logic, content modification) → proxy frozen-copy applies, needs full CC-session-restart.
- Bug in `src/proxy_display/`, `src/panes/`, `src/hooks/`, `src/workers/`, `src/core/monitor*`, `src/input/` → Monitor pane code, respawns on Ctrl+R, picks up current disk state.

Concrete failure (2026-04-23): During 34u debug, user reported "y-hotkey copies whole pane". Opus claimed the live Monitor ran the frozen addon copy so the new y-handler code wasn't active. User corrected: Ctrl+R respawn-pane re-imports Python modules from disk, Monitor UX is independent of proxy addon. Bug was actually real (cumulative-history serializer), the frozen-copy claim was a misleading deflection.

### Proxy Strip — Audit Before Assume

When user reports "still seeing stripped SRs" / "strip-mods count wrong" / "proxy strip isn't working as documented" — do NOT debug case-by-case. Write a scanner against `raw_payload.messages` that iterates ALL recent JSONLs and classifies every SR occurrence by marker × location. Inline python heredoc is forbidden by the tool-use rule; write the scanner to `/tmp/strip_audit.py` via Write tool, run, output to file.

Rationale (session 2026-04-19 evening): user reported unstripped task-tools-nag. First instinct was per-case debug. Systematic audit found **24,282 SRs bypassing strip across 6 recent logs in under 2 min** — all in `tool_result.content` string which the strip functions didn't recurse into. Without the audit, the 24,282 figure stays invisible; individual debugging cycles would have taken hours.

Audit scan locations (every SR occurrence in a message):
- `text_block` — top-level user-message text block
- `tool_result_str` — string inside `{type: tool_result, content: "..."}`
- `tool_result_nested_text` — `{type: text, text: "..."}` nested inside tool_result.content list
- `plain_string` — message.content is a plain string (not blocks)

Expected coverage: if the strip function only handles `text_block`, EVERY SR in `tool_result_*` leaks. Audit confirms or refutes this per-location.

### Strip Audit Scripts — must mirror Proxy targeting

When writing a `dev/tool_use_analysis/` audit that asks "what did the proxy strip / what survived", the scan logic in the audit MUST use the same pattern-matching primitives as the proxy strip code does. Substring search where the proxy uses anchored regex over-reports — it finds code-literal mentions, JSON-stringified content dumps, and embedded markdown discussions of tags that the proxy correctly never targeted. Anchored regex where the proxy uses substring under-reports.

Reference matrix (current proxy logic — verify against `src/proxy/strip_*.py` if changed):

| Tag | Proxy match logic | Source |
|-----|-------------------|--------|
| `<system-reminder>` | Line-anchored regex `(?m)^<system-reminder>(.*?)</system-reminder>` (DOTALL) | `src/proxy/strip_sr.py:_apply_sr_strip` |
| `<persisted-output>` Preview | Block-internal regex with `\nOutput too large` + `\n+Preview \(first ` anchors | `src/proxy/strip_po.py:_PO_PREVIEW_RE` |
| `<task-notification>` | Line-start substring matching of `<task-notification>` | `src/proxy/strip_*.py` (TN handling) |
| `<new-diagnostics>` | Always wrapped in `<system-reminder>` (pyright-diagnostics template); standalone `<new-diagnostics>` is a code-literal in dumped content, NOT a real CC injection | `src/proxy/strip_sr.py:_SR_TEMPLATES` |

**Workflow when building or extending an audit:**

1. Identify which tag(s) / templates the audit will scan for.
2. Open the corresponding strip function in `src/proxy/strip_*.py`. Read the exact regex / substring it uses.
3. Mirror that exact primitive in the audit's scan loop. Document the parity in a comment referencing the proxy file:line.
4. After running on a real session, manually inspect the first 10 hits per tag-type. False-positives mean the audit's match primitive does NOT mirror the proxy correctly — fix before trusting numbers.

**Asymmetry within the current `tag_presence_audit.py`:** SR uses the line-anchored mirror correctly. TN/ND/PO use simple substring `if tag_str in text:` — over-reports for tag-literal mentions in tool_result content. True-positive pattern requires the tag at line-start in actual API content, not as code-quoted documentation. Anchored TN/ND/PO detection is a follow-up if the false-positive rate proves problematic.

Concrete failure (2026-04-28, first iteration): `tag_presence_audit.py` used `text.find('<system-reminder>')` substring scan against `1777294641.jsonl`. Reported 14 SR bypasses, 12 of which were code-literal mentions in dumped session JSONL content (REQ#98 had `if "<system-reminder>" in text:`, REQ#137-141 had file dumps with literal SR mentions). Anchored regex switch → 14 → 1 occurrences, matching reality.

Concrete failure (2026-04-28, R2 audit 1777320131): flagged 2 ND-bypasses at REQ#29 + REQ#42 msg[56]. Both turned out to be Bash `tool_result_str` content containing the text of a previous audit MD which displayed `<new-diagnostics>` as a Mock-Example documentation block. Not real CC-injected diagnostics. Same pattern surfaced earlier for `[SYSTEM NOTIFICATION]` substring matches inside `strip_sr.py` source code reads.

### Two-Regex Mismatch in Strip Code

When proxy code has BOTH a "mutate" regex (the one that actually strips/modifies content) AND a "record" regex (the one that populates `stripped_msg_removed` or similar log field for diagnostics/UI), they MUST share the same anchoring semantics. A mismatch leads to the record-regex capturing content that the mutate-regex never touched — the log over-reports, downstream tools (Monitor display, audit scripts) render or classify correctly-present content as if it were removed.

Rule:
- When authoring or reviewing proxy strip code that pairs "mutate" + "record" regexes (or "detect" + "count"), verify both regexes share anchors and flags. Document the pair's invariant in a comment.
- When fixing a single regex, grep for parallel regexes in the same module and check them as a class.
- Every strip rule test should include a "mid-line literal" case to expose this bug class.

Concrete failure (2026-04-22): `_STANDALONE_SR_RE` in `strip_sr.py:7` uses `(?m)^<system-reminder>` (line-start anchored). `_find_system_reminder_blocks` in `payload_helpers.py:16` used `<system-reminder>` (no anchor). msg[10] in REQ#6 had 6 mid-line literal tag mentions + 1 real line-start SR at end: actual strip removed 517c, log stored 12,251c as "removed". Monitor rendered the 12,251c range as DIM_YELLOW_BG stripped — misleading visual of "huge area stripped" when 11,734c were still in the payload. Fix `5add1bf` added `(?m)^` anchor to match strip semantics. Same-class issue surfaced in `_pm_pat` (`rules.py:51`, plan-mode) — same fix pattern.

### Log-Data Immutability Across Fixes

Fixes to proxy logging code (which fields to write, how) are NOT retroactive. Existing JSONL entries carry the pre-fix field values unchanged. Downstream consumers (Monitor pane, audit scripts) reading historical entries continue to display the pre-fix data correctly-but-stalely, even after the fix is merged.

When a fix targets proxy logging behavior:
- State explicitly in the bead/recap "live-verify requires proxy restart + new triggering request". Existing log entries are not a regression indicator for this class of fix.
- When user reports "fix didn't work, still see X": first check whether the data they're looking at was logged pre-fix. `jq -r '.timestamp' <jsonl> | head -1` gives the log start; compare to fix commit time.

Concrete failure (2026-04-22): `(?m)^` anchor added to `_find_system_reminder_blocks`. Monitor kept showing 12,251c stripped region on historical REQs because their JSONL entries were logged with the buggy regex. Only new entries written after proxy restart use the fixed regex. Combined with the Proxy Addon Live-Copy rule above (proxy needs full CC-session restart to pick up rules.py changes), the user pressed Ctrl+R, saw old data on old entries, concluded "fix doesn't work" — neither restart had happened.

### `raw_payload` ist post-strip — für Original-Content die `stripped_*`-Felder nutzen

Das Feld `raw_payload` im Proxy-JSONL enthält nicht das Original-Payload das Claude Code geschickt hat, sondern den Zustand nachdem `apply_modification_rules` alle Strip-Regeln durchgelaufen ist. Der Addon übergibt in `addon.py:95` das `modified_payload` an `_build_entry`, und dort wird es auf `raw_payload` gemappt (`logging.py:51`). Daraus folgen zwei Fallen bei der Strip-Diagnose.

Erste Falle: ein Grep nach einem Marker im `raw_payload` kann ihn nicht mehr finden, obwohl er ursprünglich im Payload war — er wurde einfach rausgestrippt bevor er ins Log kam. Wer daraus schließt „der Marker kam gar nicht an", liegt falsch.

Zweite Falle: ein Grep findet einen Marker IM `raw_payload`, obwohl in `modifications` ein zugehöriger `stripped_*_sr`-Eintrag steht. Das passiert weil der Strip template-basiert arbeitet (siehe `strip_sr.py:_SR_TEMPLATES`) — ein SR-Block wird nur entfernt wenn sein innerer Text mit einem der registrierten Template-Identifier BEGINNT. SR-Blöcke deren innerer Text mit was anderem anfängt (z.B. der claudeMd-Context-Block der mit „As you answer the user's questions..." startet) rutschen durch und landen unverändert im `raw_payload`, auch wenn ein anderer claudeMd-SR in einer späteren Message gestrippt wurde und den `stripped_claudemd_sr`-Eintrag ausgelöst hat.

Wer den Original-Content sehen oder analysieren will, muss NICHT in `raw_payload` grepen, sondern in die parallel gespeicherten Felder schauen. `stripped_msg_indices` listet die Indizes der veränderten Messages. `stripped_msg_originals` enthält pro Index den summarized Original-Content vor dem Strip. `stripped_msg_removed` enthält pro Index die konkreten Text-Fragmente die rausgenommen wurden. Für den system-Block gibt es zusätzlich `original_system2_text` falls der durch `replaced_system_prompt` getauscht wurde. Die Warnings-Pane und die Proxy-Display-Pane im Monitor lesen genau aus diesen Feldern — deshalb kann der User überhaupt gestrippten Content im Monitor sehen.

Für echte Strip-Verifikation baut man entweder einen Unit-Test mit konstruiertem Input oder ein Script das die Rule-Kette isoliert gegen ein Sample-Payload laufen lässt. Grepen im `raw_payload` alleine reicht nicht.

### Worker-Prompts über Strip-Logik — Unicode-Platzhalter statt echter Tags

Wenn du einem Worker einen Prompt schreibst in dem es um Strip-Logik, System-Reminder oder Diagnostics-Tags geht, benutz im Prompt-Text Unicode-Platzhalter statt der echten Tag-Syntax: `⟨SR⟩` / `⟨/SR⟩` für `<system-reminder>`, `⟨TN⟩` / `⟨/TN⟩` für `<task-notification>`, `⟨ND⟩` / `⟨/ND⟩` für `<new-diagnostics>`. Erklär die Konvention am Anfang des Prompts damit der Worker weiß dass die Platzhalter den echten Tags entsprechen.

Der Grund: unser eigener Proxy strippt `<system-reminder>`-Blöcke die am Zeilenanfang stehen und mit einem der zehn registrierten Template-Identifier beginnen (siehe `strip_sr.py:_SR_TEMPLATES`). Wenn dein Prompt zufällig einen solchen Block als Beispiel zitiert — etwa einen vollständigen `<system-reminder>`-Block mit „The task tools haven't been used recently..." als Einleitung — greift der Strip beim Versenden des Worker-Spawn-Requests. Der Worker bekommt dann einen leeren oder kaputt gekürzten Prompt und kann seine Aufgabe nicht ausführen.

Mit Platzhaltern passiert das nicht, weil die Template-Identifier literale Strings sind und `⟨SR⟩` keinem davon entspricht. Der Worker schreibt in seinem eigenen Code trotzdem normale `<`/`>`-Zeichen, weil die Platzhalter-Konvention nur für den Prompt-Text gilt, nicht für den Code den der Worker produziert.

Failure 2026-04-21: proxy-strip-fix worker wurde zweimal in Folge mit EMPTY prompt gespawnt. Die ersten zwei Versuche hatten literale `<system-reminder>`-Beispiele im Prompt-Text — unser Proxy hat den Prompt beim Versand gestrippt bevor er beim Worker ankam. Erst der dritte Spawn mit `⟨SR⟩`-Platzhaltern war erfolgreich.

### Cache Response Data Source (CRITICAL)

When investigating cache behavior (CR/CC numbers), the PROXY LOG is NOT the source.

| Surface | File | Contains |
|---|---|---|
| Proxy JSONL | `src/logs/api_requests_*.jsonl` | `raw_payload` + `sent_meta` — what CC SENT + what the proxy FORWARDED. `sent_meta.sent_cache_read` / `sent_meta.sent_cache_creation` are always `null` in practice (proxy does not see the response). |
| Session JSONL | `~/.claude/projects/<encoded_path>/<session>.jsonl` | API RESPONSE — `message.usage.cache_read_input_tokens`, `cache_creation_input_tokens`, `input_tokens`. THIS is where cache numbers live. |

**Rule:** When the user reports "cache rebuild" or "rules didn't cache", FIRST open the Session JSONL, NOT the proxy log. Proxy log is only useful for "was the prefix byte-identical?" questions (`tools_bytes_hash`, `prefix_hash_bp2_tools`). Actual cached/created token counts are response-side.

**Frame the limitation up front:** "proxy log cannot show cache numbers, those live in Session JSONL. Proxy data can only tell us if the prefix was byte-stable."

Concrete failure (2026-04-15): During e1u cache investigation I queried `sent_meta` and reported "tools_bytes_hash stable, no rebuild visible" — but this silently omitted that sent_meta NEVER carries response data. User had to clarify the rebuild was in Token Pane = response side.

### Cache Rebuild Investigation

**TTL is NOT an explanation for mid-session rebuilds.** During an active session requests fire every few seconds — TTL is permanently refreshed. A mid-session rebuild's cause is ALWAYS a prefix change, NEVER TTL.

Possible mid-session rebuild causes:
- `shared-rules/` file edit → proxy reloads rules → `sys[2]` content changes
- `proxy_rules.json` edit → proxy reloads config → `sys[2]` content changes
- CC context editing / message rewriting → message prefix hash changes
- Worker `tmux_spawn` → unclear if side effects (investigate case-by-case)

**Investigation first step:** check whether `shared-rules/` or `proxy_rules.json` mtime changed. 10-second verification, most common cause.

### Cache Prefix Order

tools → system → messages. Verified from `PromptCaching2.md:55`. Not the other way around. Don't guess.

### Cache-Aware per-REQ Overrides (Anthropic API)

Source: `Monitor_CC/sources/AdaptiveThinking2.md` ("Prompt caching" section), `Monitor_CC/sources/PromptCaching2.md`.

When the proxy modifies an outbound payload to control thinking depth or cost on specific request patterns:

- **`thinking.type` mode switches break messages cache.** Anthropic explicitly documents that switching between adaptive ↔ enabled ↔ disabled invalidates prompt cache breakpoints for messages (system + tools stay cached). On a 132k-token messages context this is ~$0.59-$0.76 in extra cost per toggle (132k × $4.50/M cache-write delta vs cache-hit-rate). Toggle ON + toggle OFF per pattern detection = ~$1.20-$1.50 per spawn cycle. The "cure" is 23-30× more expensive than a single abort cascade ($0.04).

- **Per-REQ API parameters do NOT break cache.** `max_tokens`, `temperature`, `top_p`, `top_k`, `output_config.effort` are per-call params, not part of the cached prefix (which hashes system + tools + messages content). Vary them freely between REQs without cache penalty.

- **On Claude Opus 4.7, manual `thinking: {type: "enabled", budget_tokens: N}` is rejected with 400 error.** Adaptive-only mode. Hard-cap on thinking via `budget_tokens` is unavailable — use `max_tokens` (caps total output incl. thinking) plus `output_config.effort: "low"` (soft-biases toward less thinking, cache-safe).

- **Cap-pattern for post-spawn-ack REQs:** detect via last-assistant `tool_use` block where `name == "Bash" AND input.run_in_background == True AND input.command matches ^sleep\\s+\\d+`. Override: `output_config.effort = "low"`, `max_tokens = 2000`, leave `thinking.type = "adaptive"` (no mode switch). Modification tag `capped_post_sleep`. Implemented in `src/proxy/rules.py` (commit cd459dd).

**When to use mode-switch vs per-REQ override:**
- Per-REQ override (preferred): any pattern that should reduce thinking cost without changing model reasoning mode. Caching preserved across the override REQ and after.
- Mode-switch (only if model genuinely should NOT reason): acceptable cost penalty. Document the cache-rebuild cost in the decision file. Rare — most patterns are better served by `effort=low + max_tokens`.

### Marker File Lifecycle (2 Mutation Points)

The per-project proxy marker file (`.proxy_session_<SESSION_ID>`) has TWO critical mutation points, not one:

1. **WRITE on proxy start** — a parallel session in the same project OVERWRITES the marker with its own `log_id`
2. **DELETE on proxy cleanup** — the exiting session removes the marker

Both must be guarded:
- Write: only overwrite if the existing port is not listening (`lsof` check)
- Delete: only delete if the marker's `log_id` matches own `LOG_ID`

Concrete failure (2026-04-16): First fix guarded only DELETE. Hi-test session overwrote the marker on START, then correctly deleted its own marker on exit → main session's marker was gone. Second fix added `lsof` guard on WRITE.

**General pattern:** When a shared-state file (marker, lock, config) causes a bug, always enumerate ALL code paths that mutate it — not just the one that triggered the symptom.

### `/tmp/.monitor_cc_proxy_<SESSION_ID>` — nicht manuell löschen

Neben der projekt-internen `.proxy_session_<SESSION_ID>` schreibt der Proxy einen zweiten Marker nach `/tmp/.monitor_cc_proxy_<SESSION_ID>`. Beide Marker enthalten Port und Log-ID. Das `iterative-dev`-Plugin liest den `/tmp/`-Marker beim Worker-Spawn, um neu gespawnte Worker an den laufenden Proxy anzubinden (HTTPS_PROXY env, Zugriff aufs Worker-Log-Dir).

Der Marker ist komplett proxy-owned. `claude_proxy_start.sh` schreibt ihn beim Start (mit Port-Guard gegen parallele Sessions) und löscht ihn beim `trap EXIT INT TERM` wieder (mit Port-Guard damit keine fremde Session getroffen wird). Kein anderer Prozess soll ihn anfassen — auch nicht der tmux-Viewer, auch nicht Opus beim „Monitor neustarten".

Wird der Marker manuell gelöscht während die Main-CC-Session noch läuft, hat das eine üble Nebenwirkung: nachfolgende `worker_spawn`-Aufrufe finden den Proxy nicht mehr und spawnen den Worker silent ohne Proxy. Der Worker läuft dann als nackte Claude-Code-Instanz — keine Rule-Injection, kein `effort=max`, kein Tool-Strip, kein Worker-Log im Proxy-Log-Dir. Das ist unsichtbar und kostet später viel Zeit bei der Fehlersuche.

Wenn die Worker-Proxy-Pane im Monitor „no proxy data yet" zeigt, ist die Diagnose in dieser Reihenfolge: erst mit `ls /tmp/.monitor_cc_proxy_*` prüfen ob der Marker existiert, dann mit `ls src/logs/api_requests_worker_<name>_*.jsonl` ob für den Worker überhaupt ein Log entstanden ist. Wenn der Marker fehlt und die Main-CC-Session noch lebt, wurde er manuell gelöscht — dann hilft nur die Main-CC-Session neuzustarten damit der Proxy den Marker neu schreibt.

Failure 2026-04-20: Marker beim „restart monitor" manuell gelöscht, die nächsten zwei Worker liefen silent ohne Proxy. Bead Monitor_CC-889.

### Per-REQ Serializer Delta-Scope

In Monitor_CC proxy logs, each `entry.messages` array contains the CUMULATIVE message history of that REQ — all messages from session start up to and including the new ones added in this specific REQ. This is how Claude Code conversations work: every API request carries the full history.

For any serializer that takes a per-REQ key and produces text for display/copy/export, ALWAYS slice messages via `diff_from_prev.first_diff_index`:

```python
diff = entry.get('diff_from_prev') or {}
start = diff.get('first_diff_index', 0) if diff else 0
if start < 0:
    start = 0
for msg_idx, msg in enumerate(entry.get('messages', [])[start:], start=start):
    ...
```

This yields ONLY the messages that are new in this REQ — matching what the REQ visually represents (msg_count delta, Δmsgs header). The `enumerate(..., start=start)` keeps msg_idx labels aligned with original entry positions.

Applies to: `_serialize_proxy`, `_serialize_worker_proxy`, any future pane-to-clipboard or pane-to-export function operating on proxy JSONL entries. Same delta-scope pattern used by `classify_tags` in `src/proxy/strip_vocab.py` and `_check_tags` in `dev/tool_use_analysis/strip_audit.py`.

Without the slice, a click on REQ#20 copies ~770kB (full history), a click on REQ#40 copies ~810kB — visually indistinguishable from "copies whole pane content".

Concrete failure (2026-04-23): commits ed8afff + e31ac57 shipped y-hotkey feature with `_serialize_proxy` iterating cumulative `entry['messages']` without delta-slice. User clicked REQ + pressed y, saw 700+kB in clipboard, reported as "copies whole pane". Fix in a1825b7 added the delta-slice. Worker audit confirmed the other 7 pane serializers already scope correctly by key (operate on per-item lists, not cumulative histories) — only the 2 proxy serializers had the bug.

## Analytics Conventions

### Ratio Direction (Units)

When specifying ratio computations for workers or dev scripts — ALWAYS state the unit explicitly.

- "chars per token" and "tokens per char" are reciprocals with very different-looking numbers (3.68 vs 0.272). Neither is inherently right, but mixing them in one analysis guarantees a correction round.
- Project convention: **chars/token** as primary unit (matches the 3.68 anchor and `pipe05_proxy_cache.md` documentation).
- In worker prompts for analytics tasks, state: "primary unit: chars/token. Formula: `chars / tokens`. Report median/mean/stddev in chars/token."
- In raw data tables: column header `ratio (c/tok)`, not just `ratio`.

Concrete failure (2026-04-17): Worker computed `CC / Δmsg_chars` (tokens/char) while anchor was chars/token. Caught in review, one worker_send correction.
