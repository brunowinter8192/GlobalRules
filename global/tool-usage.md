# Tool Usage

## No Bash for File Creation

**Rule:** NEVER use `cat > file << 'EOF'` or `echo >` for creating files. Always use the Write tool.

**Why:** Bash heredocs can leak shell context into file content (e.g., `EOF 2>&1 | head -10` appended to a .gitignore). The Write tool is atomic and safe.

## Context Window Hygiene — NO Verbose Output (CRITICAL)

**Problem:** Large tool outputs (crawl logs, build output, test runners, TaskOutput) flood the context window. One verbose TaskOutput can burn 10k+ tokens of irrelevant noise.

**Rule: ALL tool output goes through files, NEVER directly into context.**

**Workflow:**
1. **Run command** → redirect output to file: `command > /tmp/output.md 2>&1`
2. **Read result** → use `grep`, `tail`, `head` on the file — extract ONLY what you need
3. **Present** → show the user a concise summary, not raw output

**Applies to:**
- `TaskOutput` — NEVER use `block=true` with verbose tasks. Let it write to file, then grep the file.
- Dev scripts — ALWAYS redirect: `./venv/bin/python script.py > /tmp/script_output.md 2>&1`
- Background tasks — read the output FILE with targeted grep/tail, not TaskOutput
- Build/test runners — same pattern: file first, grep second
- MCP tool calls with large responses — summarize, don't dump

**Concrete pattern:**
```bash
# WRONG — dumps everything into context
./venv/bin/python dev/crawling_suite/03_test.py

# RIGHT — output to file, then extract what matters
./venv/bin/python dev/crawling_suite/03_test.py > /tmp/03_test_output.md 2>&1
tail -20 /tmp/03_test_output.md  # just the summary
```

**TaskOutput pattern:**
```bash
# WRONG — block=true streams entire log into context
TaskOutput(task_id="xxx", block=true)

# RIGHT — wait for completion notification, then read the REPORT file
# Dev scripts already write MD reports — read THOSE, not raw stdout
cat dev/crawling_suite/03_reports/latest.md
```

**The test:** Before ANY tool call that might produce >50 lines of output — redirect to file first. No exceptions.

**MCP Tools with exploration scope (explore_site, search_web etc.):**
- MCP tools are for QUICK checks only (max_pages=20, single queries)
- Full exploration/crawling → run the underlying Python script directly with output redirect
- NEVER use MCP tools for large-scale operations — they dump everything into context and block the conversation

**Bash commands without redirect:**
- NEVER run `./venv/bin/python script.py` without `> /tmp/file.md 2>&1`
- NEVER run `cat` on a file that might be large — use `head`, `tail`, `grep`
- NEVER use TaskOutput(block=true) — wait for completion notification, then grep the output file

**Why:** Context window is the most precious resource. Every wasted token = less room for actual work. One verbose dump can cost an entire conversation turn. This is NON-NEGOTIABLE — a single violation can burn 10k+ tokens of irrelevant crawl logs, build output, or test runner noise.

## Stop After 2 Failed Tool Calls

**Problem:** Going in circles when tool calls fail.

**Rule:**
- When **2 tool calls in a row** fail or don't deliver the desired result
- **STOP IMMEDIATELY**
- Clearly explain the problem to the user
- Ask: "How should I solve this?" or "Where can I find X?"
- **NO** further trial and error without user input

**"Quellen" = External Research, NOT More Bash:**
- When user says "welche Quellen brauchst du?" → answer with CONCRETE sources: GitHub Issues, Web docs, man pages, source code files
- NEVER respond by running the same command again in a different wrapper
- After 2 failures: the answer is RESEARCH (Web/GitHub search, read source code, read docs), not RETRY
- Concrete failure (2026-03-26): User asked 3x for needed sources. Response was more bash commands instead of naming concrete GitHub issues, tmux docs, or web searches to consult.

**Same Error, Different Wrapper = Same Bug:**
- When the same URL fails via MCP tool AND via direct Python call AND via another tool variant: the problem is NOT the wrapper.
- After 2nd failure: analyze the error pattern instead of trying another calling method.
- Ask: "What is the common denominator across all failures?" (e.g., is_garbage_content() rejecting the content in ALL code paths)
- Concrete failure (2026-03-15): CookieYes URL failed 3 times (scrape_url, scrape_url_raw, direct Python) before understanding that is_garbage_content() was the common cause.

**Principle:** Efficiency over trial-and-error.

## Tool Failure → Immediate Action (CRITICAL)

**Problem:** Tool call fails silently (missing dependency, server down, wrong config), but Claude continues with a workaround or fallback instead of fixing the root cause or reporting immediately.

**Rule:**
- When a tool call fails or returns an error: **report to user IMMEDIATELY** in the same response
- Then either: (a) **fix the prerequisite yourself** (start server, install dep, fix config) and retry, OR (b) **stop and wait for user input** if the fix is outside your control
- NEVER silently fall back to a different approach without telling the user the primary tool failed
- NEVER ask "should I start X or work around it?" — if you CAN fix it, fix it. If you can't, say so.

**Concrete failure (2026-03-28):** RAG `search` failed with "llama-server not found" (embedding server not running). Instead of starting the server or reporting the failure, Claude fell back to `read_document` (no embeddings needed) and only mentioned the issue when presenting results. User had to say "starte den server wenn er nicht läuft". Should have: reported the error, started the server, then retried the search.

**Decision tree:**
1. Tool fails → report error to user in same message
2. Can I fix it? (start process, install dep, create dir) → fix it NOW, retry
3. Can't fix it? (needs user credentials, hardware, manual step) → stop, explain what's needed
4. NEVER: silently switch to plan B without disclosing plan A failed

## MCP Server Connection Lost

**Problem:** MCP server crashes silently, subsequent calls all fail with "No such tool".

**Rule:**
- When an MCP tool returns "Connection closed" or "No such tool available": the MCP server crashed.
- Do NOT fire more calls that will all fail — STOP after the first "No such tool" error.
- Recovery: inform user, suggest `/mcp` to reconnect, or fall back to direct Python calls.
- Do NOT silently switch to a different approach without telling the user the MCP server is down.
