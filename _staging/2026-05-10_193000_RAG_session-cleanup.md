# 2026-05-10 — RAG: Multi-plugin chain + skill-vs-rule precedence

## ~/.claude/shared-rules/global/tool-use.md → Hard Rule 16 (cd-Drift) — extension

### Multi-plugin Bash chain — explicit cd per plugin

When committing + publishing multiple plugins in one Bash chain, EVERY `plugin-publish` call needs its OWN explicit `cd <plugin-source-repo>` immediately before. `plugin-publish` reads the current cwd to detect which plugin to publish; without an explicit cd, the second `plugin-publish` in a chain inherits the cwd of the first and silently runs against the wrong plugin.

**Pattern (safe):**

```bash
( git -C <plugin-A-repo> commit -am "..." && \
  cd <plugin-A-repo> && plugin-publish && \
  git -C <plugin-B-repo> commit -am "..." && \
  cd <plugin-B-repo> && plugin-publish )
```

**Anti-pattern (silent wrong plugin):**

```bash
( cd <plugin-A-repo> && plugin-publish && \
  git -C <plugin-B-repo> commit -am "..." && \
  plugin-publish )                          # ← inherits A's cwd, publishes A again
```

**Symptom:** second `plugin-publish` output shows `Plugin: <plugin-A-name>` and `Everything up-to-date` instead of plugin B's name + actual push. Plugin B's cache stays stale even though its commit went through normally.

**Observed:** session 2026-05-10 RAG, recap-skill removal — first chain had `cd blank && plugin-publish` then `git -C RAG commit` then `plugin-publish` without a `cd RAG`. Result: blank published twice, RAG never. Recovery: separate `cd RAG && plugin-publish` call afterwards.

The subshell `( ... )` does NOT contain cd-leakage to the parent shell, but inside the subshell each `cd` IS sticky for the next command — so the rule is "every plugin-publish gets its own cd, no implicit re-use", not "subshell magic prevents this".

---

## ~/.claude/shared-rules/global/tool-use.md → new section: "Skill examples vs. Rules — precedence"

### Skill examples never trump active rules

When a Skill's content shows a concrete example (e.g. command with hardcoded path, specific worker spawn line, project-path substitution), the example illustrates ONE valid usage but is NOT authoritative. Rules in `~/.claude/shared-rules/` always take precedence.

**Test before following a skill example:** does this example respect all active rules? If no, modify the example to match the rule before executing. The rule wins.

**Concrete case (session 2026-05-10):** `web-research/SKILL.md` "Permanent Capture Workflow" Step 4 had:

```bash
worker-cli spawn cleanup-<collection_lower> /tmp/spawn-<worker_name>.md \
    /Users/brunowinter2000/Documents/ai/Meta/ClaudeCode/MCP/searxng sonnet
```

This violated Worker-Project-Scope (`opus/workers-1.md`): "Workers are spawned only for coding tasks IN THE CURRENT PROJECT." Following the example without checking the rule = bug. Fixed in `web-research/SKILL.md` to use `<current_project_root>` placeholder + explicit pointer to the rule.

**Rule of thumb:** when a skill example references a specific absolute path that is plugin-specific or project-specific, treat the path as a placeholder, not a literal. Substitute the value the active rules require.
