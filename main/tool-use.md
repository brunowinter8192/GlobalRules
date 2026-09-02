# Tool-Use — Orchestrator-Only

## Bash

### Worker CLI

**Worker names are globally unique.**
- A registry tracks every worker name.
- Only `spawn` requires the project path.
   - All other commands resolve the worker via the registry, to the project it was spawned in.
- For a cross-project worker, append `<project_path>` explicitly to every later command.
   - Details sit in the Worker Project Scope section of workers.md.

**`worker-cli response` is the default for reading idle workers.**
- `response` returns clean assistant text from the session JSONL.
- `capture` is the fallback for the rare case that `response` misses context.
   - `capture` returns the tmux pane, already cleaned and scoped to since the last prompt.
- Both print straight to context.
   - Never pipe them through `tail`, `head`, or `sed`, because a hook strips that anyway.

**Session name pattern.**
- The pattern is `worker-<basename(project_path)>-<name>`.
   - Project `/Users/x/Monitor_CC` with worker `inject-fixes` gives session `worker-Monitor_CC-inject-fixes`.

| Operation | CLI |
|---|---|
| List active workers (project) | `worker-cli list <project_path>` |
| List active workers (all) | `worker-cli list` |
| Check worker status | `worker-cli status <name> [project_path]` |
| Clean output since last prompt | `worker-cli capture <name> [project_path]`. With `--raw` it writes the raw pane to a file. |
| Clean last N assistant turns (JSONL) | `worker-cli response <name> [N] [project_path]` |
| Send message to running worker | `worker-cli send <name> <message>` |
| Merge worker branch | `worker-cli merge <name> [project_path]` |
| Kill worker (+ registered cross-project worktrees) | `worker-cli kill <name> [project_path]` |
| Spawn worker in worktree | `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` |
| Create cross-project worktree | `worker-cli worktree <name> <target_repo> [branch]` |
| Revive dead worker (resume CC session) | `worker-cli revive <name>` |

### Git

**Preview staged state before committing with `git-check [repo_path]`.**
- The command stages everything and reports staged, unstaged, untracked files, and hook status.
- It does not commit.
- Using it is optional review.

| Operation | CLI | Notes |
|---|---|---|
| Push (NON-plugin repo) | `git -C <repo_path> push` | Falls back to `-u origin <branch>` if no upstream exists. Use `plugin-publish` instead if `.claude-plugin/plugin.json` exists. |
| Push with upstream (NON-plugin repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | For the first push on a new branch. |
| Push (PLUGIN repo) | `cd <plugin-source-repo> && plugin-publish` | One step that pushes, syncs the plugin cache, and bumps the version. Always use this for any repo with `.claude-plugin/plugin.json`. A plain `git push` on a plugin repo is not allowed. |

### RAG CLI

**RAG queries are ALWAYS written in English, regardless of conversation language.**
- Issue the search command directly.
- No prior `rag-cli server start` is needed.

**`delete` removes everything the scope covers.**
- It removes the matched chunks and their rows in the `indexed_files` manifest.
- It also removes the on-disk source files under `data/documents/<collection>/`.
- `--collection` is required, and alone it deletes the whole collection.
- `--document` optionally narrows the deletion to one document.
   - That removes the document's chunks, its manifest row, and its `.md` and `.json` files.
- `--document` without `--collection` errors.

**`index` is the inverse of `delete` over the same scope.**
- It chunks, embeds, and stores `.md` files from `data/documents/<collection>/`.
- `--collection` is required, and alone it indexes every `.md` in the collection directory.
- `--document` optionally indexes just that one file.
- Unchanged files are skipped by default, detected via content hash.
   - `--force` re-embeds everything.
- `--document` without `--collection` errors.

**`search` finds the hit and `read_document` pulls the context around it.**
- `read_document <coll> <doc> <chunk> --before N --after M` returns the chunk plus its neighbors.
   - The useful detail usually sits in those neighbors.

**Every chunk you build on gets expanded with `read_document` first.**
- Building on a chunk means writing an artifact from it, stating a claim from it, or acting on it.
- Before any of that, run `read_document` with `--before` and `--after` on that chunk.
- A bare `search` hit is never a sufficient basis to act on.
- Believing you already have all the information is never grounds to skip the expansion.
- Building on N chunks means N expansions before you act.

**Miss handling.**
- On a result with zero chunks, reformulate the query at least twice before falling back.
   - The fallback is a direct Read or a bash grep.
- On a partial hit, run `read_document` around the hit's chunk index instead of re-querying.

| Operation | Command |
|---|---|
| List collections | `rag-cli list_collections [--filter PATTERN]` |
| List documents | `rag-cli list_documents <collection> [--document PATTERN] [--exclude PATTERN] [--filter PATTERN]` |
| Search | `rag-cli search <query> <collection> [--document PATTERN] [--exclude PATTERN]` |
| Read context | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Delete | `rag-cli delete --collection <name> [--document <doc>]` |
| Index | `rag-cli index --collection <name> [--document <doc>]` |

### GitHub Issues (gh-cli) — Cross-Session Context

**Derive `<owner>` and `<repo>` from the git remote.**
- `git remote get-url origin` returns `github.com:<owner>/<repo>.git`.
   - Pull both values from there.
- Hardcoding them is not allowed.

**Open issues are the default listing.**
- `gh-cli list_issues` shows open issues by default.
   - Closed issues appear only with `--state closed` or `--state all`.
- Pull requests are filtered out.

**Finding the number.**
- `gh-cli list_issues <owner> <repo>` lists open issues one per line as `#N [OPEN] title`.
   - Match by title and pass the stable issue number.

#### What an Issue IS

**An issue is a lean entry point into a topic.**
- The body names the topic and where its content lives, because the content itself lives elsewhere.
- Source code carries the current architectural state, so read the code.
- `process-docs/<area>/` carries the process history, meaning investigations, measurements, and reasoning.
- `<package>/DOCS.md` carries the module map.
- The RAG `<Project>-reference` collection carries external sources like vendor docs and papers.
- Resuming works via RAG search on the docs and reference collections plus reading the code.

**Issues are created at exactly two points.**
- The first point is when the user asks mid-session.
- The second point is Recap, for whatever is still open at session end.
- Between those there is no autonomous issue-keeping.

#### Issue Format

```
<Title: ONE word>

Goal:
- <end state>
- <end state>

Area: <area>  (→ process-docs/<area>/, dev/<area>/)
```

**The title is ONE word, and it names the thing, never the action.**
- `Wohnungsmängel`, `Bügeleisen` and `Hausarzt` are titles, while `Zahnarzt in Frankfurt finden und Kontrolle 2026` is not.
- An action in the title ages the moment the action is done, and the thing stays the thing.
- Where one word genuinely cannot name it, the title stays a noun and never becomes a verb.

**The body carries the GOAL, meaning the end state, and nothing else.**
- The end state is what makes closing the issue a yes-or-no question.
- It is one bullet where the state is single, and several bullets where it genuinely has several parts.
- A bullet is a state, not a step, so it says what will be true and never what gets done.
- Everything that is not the end state stays out, meaning history, reasoning, status, next actions, and dates.
   - Those age between sessions, and a body that ages is a body nobody trusts.

**The body carries no file paths and no resume instruction, because the area name leads to everything else.**
- Everything under `process-docs/<area>/` and `dev/<area>/` is found via RAG plus folder browsing.
- Code is read directly.
- DOCS.md and reference documents are found by searching their collections.
- The body is written once.
   - A rewrite happens only when the goal or the area changes.

#### Resume Pattern

**Picking up an open issue in a new session follows three steps.**
1. Read the issue via `gh-cli get_issue <owner> <repo> <number>`, because the body carries the area.
2. RAG-search `<Project>-docs` scoped with `--document 'process-docs/<area>/%'`, and `<Project>-reference` for external sources.
3. Browse the `process-docs/<area>/` and `dev/<area>/` folder listings for entries RAG missed.

- The issue does not contain narrative, because the sources do.

#### Issue-Close

**Close with `gh-cli update_issue <owner> <repo> <number> --state closed`.**
- Close proactively when the issue's code is merged and a live check shows the behavior works.
   - Do that in the same flow instead of waiting for the user to ask.
- Nothing is posted to the issue at close, because process-docs carries the summary.
- If the issue defines a verification test, run the test before closing.

| Operation | CLI |
|---|---|
| List open issues | `gh-cli list_issues <owner> <repo>`. Open is the default state. |
| List closed issues | `gh-cli list_issues <owner> <repo> --state closed` |
| Read issue body | `gh-cli get_issue <owner> <repo> <number>`. The body is the text after the `---` separator in the output. |
| Create issue | `gh-cli create_issue <owner> <repo> "<title>" --body "<desc>" [--labels a,b]` |
| Update issue body (area change only) | `gh-cli update_issue <owner> <repo> <number> --body "<full updated body>"`. The call replaces the full body. |
| Close issue | `gh-cli update_issue <owner> <repo> <number> --state closed` |
| Reopen issue | `gh-cli update_issue <owner> <repo> <number> --state open` |

### show — open a file for the user

**Open a file in the user's default macOS app so the USER can see it.**
- Use it when the user asks to be shown a file, like "öffne mir den Report" or "show me X".
- The trigger is the user's intent to look at it, and never the file type.

**Use `show` only when the user wants to LOOK at a file.**
- For your own inspection like analysis, code review, or grep, use Read or Bash.

**A file already opened with `show` stays open.**
- One `show` at first display holds for the whole session.
- The open app picks up later edits in place.
   - Re-running `show` after each edit is therefore redundant.

| Operation | Command |
|---|---|
| Open one file | `show <path>` |
| Open multiple files | `show <p1> <p2> ...` |
| Relative path | `show ./report.md` |
| Home path | `show ~/Desktop/foo.png` |
