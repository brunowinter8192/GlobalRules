# 2026-05-07 — TRADING: worker skill activation + shared-rules git correction + plugin clarifications + no-anecdote rule

## `~/.claude/shared-rules/opus/workers-1.md` → Phase 1 Prompt Structure ("MUST include")

Add new bullet to the MUST-include list:

- **Plugin skill activation (when applicable)** — if the worker needs a plugin skill that isn't part of `iterative-dev` (e.g. `cleanup-and-index` from searxng, `agent-rag-search` from rag), the prompt MUST direct the worker to call `Skill(skill="<name>")` as its FIRST action, before any investigation step. All plugins are globally available — there is no `enabledPlugins` to maintain — but skill *content* is not auto-loaded; the worker must explicitly request it. Without this, the worker falls back to whatever instructions are inline in the prompt, missing the canonical methodology that lives in the skill.

## `~/.claude/shared-rules/global/environment.md` → Section "~/.claude/ is NOT a Git Repo"

Replace the section. Current text says `~/.claude/` is local-only and not git-tracked. That's wrong for `shared-rules/`, which IS a git repo with its own remote (verified 2026-05-07: `git -C ~/.claude/shared-rules rev-parse --git-dir` returns `.git`, 20+ tracked files).

New version:

> ## `~/.claude/` Tracking Status
>
> Most of `~/.claude/` (settings, plans, plugins cache, projects, beads) is local-only — not git-tracked, not committed, not pushed. Don't flag "not committed" for files in those subdirs — that's expected.
>
> **Exception:** `~/.claude/shared-rules/` IS its own git repo with its own remote. Rule edits there are tracked; commit and push apply normally. During a live session, follow the staging workflow under `~/.claude/shared-rules/_staging/` instead of editing rule files directly — the proxy rule loader watches mtime and any edit invalidates the rule cache, costing roughly the current context as `cache_creation_input_tokens` on the next request.

## `~/.claude/shared-rules/situational/plugins.md` → after the "Plugin Catalog" table

Append a clarifying paragraph (the catalog already says "all plugins global", this nails the consequence for workers):

> **Skills are global but not auto-loaded.** A worker spawned in any project has access to every installed plugin's tools/skills/agents/commands — there is no `enabledPlugins` filter to maintain (an outdated mechanism, no longer used). But skill *content* loads only on explicit `Skill(skill="<name>")` call. When dispatching a worker that should follow a specific skill's methodology, name the skill in the prompt's first instruction. See `opus/workers-1.md` Phase 1 Prompt Structure.

## `~/.claude/shared-rules/global/communication-2.md` → after the "Self-honesty" section

Add a new section near the end of the file:

> ## No Anecdotal Annotations in Rules
>
> Don't add "Concrete failure (DATE): ..." or "Past failure (DATE): ..." paragraphs to rule files, skill docs, or worker prompts. Rules state what to do, not what went wrong on a specific past session. The historical context doesn't help the next reader and bloats the file with low-density narrative. If a past failure motivated the rule, the rule's wording itself encodes the learning — no annotation needed.
>
> If a past session's anecdote feels structurally informative (e.g. it explains a non-obvious mechanism), promote that information into the rule body proper as a statement of fact, not as a dated aside.

## `~/.claude/shared-rules/worker/dev-convention.md` (or new `worker/long-running-jobs.md`) → new section "Timer duration must match the work"

Worker rule (general — not tied to one skill). Promote this from the cleanup-and-index skill update so all workers running long-async jobs know it:

> ## Timer Duration Must Match the Work
>
> When watching a long-running background process (indexer, MinerU run, build, training job), the timer duration of any `Bash(run_in_background=true)` poll MUST approximate the expected remaining wallclock — not a default like 60–120 seconds. A 29-minute job polled at 120s = 14 cascade events; each completion arriving while an API stream is open cancels the stream and bills input + cache-read.
>
> Two acceptable patterns:
>
> 1. **Last step before completion checklist:** ONE backgrounded call with an internal sleep loop that exits when the work is done. Zero polling, ONE completion at the very end:
>    ```bash
>    ( while pgrep -f '<pattern>' > /dev/null 2>&1; do sleep 30; done; echo DONE ) &
>    wait
>    ```
>    The `sleep 30` inside is normal blocking sleep, NOT a tool-completion event.
>
> 2. **More steps follow:** ONE timer sized to ~80–90% of expected duration. Re-size on each subsequent timer based on remaining work.
>
> Manual one-shot status reads triggered by user request are always fine — the anti-pattern is *automated repeated short polling*.
