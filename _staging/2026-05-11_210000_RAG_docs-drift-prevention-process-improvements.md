# 2026-05-11 — RAG: docs drift prevention infrastructure session

Four process improvements surfaced this session. Apply at next session start (or whenever cache-bruch is acceptable).

---

## ~/.claude/shared-rules/opus/workers-3.md → § Merging — add cwd verification

**Pattern observed this session:** `worker-cli merge <name>` followed by `git merge docs-cleanup --no-ff -m "..."` silently placed the merge commit on `server-manager-split` branch instead of `dev`. Caused by cwd drift — the bash subprocess for an earlier worker-cli operation had landed me in the worktree (`.claude/worktrees/server-manager-split/`) without an obvious cue. Subsequent commands inherited that cwd. Result: 3-way merge of docs-cleanup landed on the wrong branch; later required a re-merge from main repo and confused state for ~30 min.

**Add to § Merging (after the Pre-Merge Clean-Check block):**

```
**cwd verification before merge (MANDATORY):**

Before ANY `git merge` (whether via `worker-cli merge` or direct `git merge`), verify the operating context:

```bash
echo "pwd: $(pwd)"
echo "branch: $(git rev-parse --abbrev-ref HEAD)"
echo "git-dir: $(git rev-parse --git-dir)"
```

Expected state for merging into dev:
- pwd: main repo root (no `.claude/worktrees/` in path)
- branch: `dev`
- git-dir: `.git` (not `.git/worktrees/<name>`)

If cwd is inside a worktree, cd back to main repo BEFORE merging. `worker-cli merge` does not auto-restore main-repo cwd — that's the caller's responsibility.

**Symptom of wrong-branch merge:** `git log --oneline -3` after the merge shows your merge commit at HEAD, but `git log --oneline dev -3` does NOT include it. If you see this, your merge landed on the wrong branch. Recovery: identify the correct branch, `git checkout dev`, re-do the merge there.
```

---

## ~/.claude/shared-rules/opus/workers-3.md → § Post-Merge Verification — add claim-vs-actual check

**Pattern observed this session:** docs-cleanup worker claimed `cc16aaf` contained whitelist additions for DATABASE/DROP/POST/SOTA. Verification grep showed the entries missing from current dev. Opus interpreted as "worker claim doesn't match reality" and accused them of fabricating. Actual cause: a later merge (server-manager-split) had branched from pre-docs-cleanup dev state and its 3-way merge silently reverted the docs-cleanup changes. The worker was telling the truth about cc16aaf; the merge mechanics destroyed the result later.

**Add to § Post-Merge Verification (after "Run `git diff HEAD~1 --name-only` ..."):**

```
**Claim-vs-current-state divergence:** If a worker claims a commit contains specific changes (e.g., file modification, new entries, function added) and current state appears not to match, BEFORE declaring "verification failure", run:

```bash
git show <claimed_commit_sha> -- <file>         # what the commit actually did
git log --first-parent --oneline -- <file>      # who touched the file since
```

A worker's commit may have been silently reverted by a later 3-way merge from a stale-base branch (classic pattern: Worker B's worktree branched from dev BEFORE Worker A's merge landed; Worker B's later merge wins via "ort" strategy and clobbers Worker A's changes without conflict). The worker's claim about their own commit can be correct AND the current state can lack those changes — these are not contradictions.

Only declare "verification failure" after confirming the commit itself doesn't contain the claim. If the commit DOES contain it but current state doesn't, the issue is post-commit merge mechanics, not worker honesty.
```

---

## ~/.claude/shared-rules/opus/workers-1.md → § Worker Phase 1: Dispatch — add Pre-Spawn Branch Sync

**Pattern observed this session:** `server-manager-split` worker was spawned while `docs-cleanup` merge was in flight. server-manager-split's worktree branched from pre-docs-cleanup dev state. When server-manager-split's branch was later merged into post-docs-cleanup dev, the ort merge strategy silently reverted docs-cleanup's whitelist additions and decision-file fixes (no conflict triggered). Pre-followup Branch Sync rule exists (workers-3 § Reusing Workers) but only fires at REUSE time, not at SPAWN time.

**Add to § Worker Phase 1: Dispatch (after the Spawning subsection, as a new subsection before "Prompt Structure"):**

```
### Pre-Spawn Branch Sync (when other workers' merges are pending)

If another worker's branch is mid-merge or about to be merged into dev, the new worker's worktree (created from current dev tip) may have a stale base. When the new worker later merges, its 3-way merge can silently revert the other worker's changes without conflict.

**Trigger:** spawning a NEW worker while ANY other worker has commits not yet on dev (or has been merged but the new worker's worktree predates that merge).

**Mitigation in the worker prompt:**

> FIRST: in your worktree, run `git -C <worktree-path> merge dev` to ensure your base includes any pending changes from other workers. Verify with `git log --oneline -5` showing the expected recent commits. THEN proceed with the task.

This is the spawn-time analog of the existing Pre-followup Branch Sync rule (workers-3 § Reusing Workers). Both prevent the same class of merge regression — stale-base 3-way merge clobbering newer changes silently.

Skip the pre-sync only when: no other workers exist, OR all other workers' commits are already on dev AND the new worker's worktree was just created (creation pulls current dev tip).
```

---

## ~/.claude/shared-rules/opus/workers-1.md → § Indexed paths — strengthen self-check phrasing

**Pattern observed this session:** Opus re-violated the freshly-sharpened "RAG-first instead of file reads" rule twice within ~20 minutes of writing the rule. Concrete violations:
- Reading `decisions/eval01_methodology.md` and `decisions/retrieval03_fusion.md` via `cat` after RAG had returned chunks
- Reading `decisions/OldThemes/eval_suite/*.md` via `cat` after RAG-features had them indexed

In both cases, the correct path was `rag-cli read_document <coll> <doc> <chunk> --before N --after M` for expansion. The mistake was jumping to direct `cat` when RAG's returned chunk didn't include the specific section needed.

**Possible strengthening — add to the self-check block:**

```
**Failure mode to specifically watch for:** when the RAG-returned chunk seems "close but not quite the section I need", the default reaction is to grab the file via `cat` or Read. WRONG. The correct next step is `read_document --before N --after M` to expand around the existing chunk. Direct read is ONLY appropriate when expansion is exhausted (`--before 5 --after 10` and still insufficient) AND the question requires full-file structure that chunks can't convey.

Self-test trigger phrase: any time the mental shape is "I have a chunk, I need MORE context" → escalate via `read_document`, do NOT escalate via `cat`. The reformulation pattern is essentially the same step that the rule already encodes; the failure pattern is reflexively skipping it.
```

This is a sharpening of the existing rule rather than a new rule. Whether worth applying depends on whether the strengthening adds signal vs adds prose without behavior change. If next session shows the same violation pattern, apply. Otherwise the existing rule is likely sufficient and the failure was just session-specific carelessness.
