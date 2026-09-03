# Tool-Use — Orchestrator-Only

## Bash

### Worker CLI

**Worker names are globally unique.**
- A registry tracks every worker name.
- Only `spawn` requires the project path.
- For a cross-project worker, append `<project_path>` explicitly to every later command.

**`worker-cli response` is the default for reading idle workers.**
- `response` returns clean assistant text from the session JSONL.
- `capture` is the reader when `status` shows `dead`.

**Session name pattern.**
- The pattern is `worker-<basename(project_path)>-<name>`.

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

| Operation | CLI | Notes |
|---|---|---|
| Push (NON-plugin repo) | `git -C <repo_path> push` | Falls back to `-u origin <branch>` if no upstream exists. Use `plugin-publish` instead if `.claude-plugin/plugin.json` exists. |
| Push with upstream (NON-plugin repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | For the first push on a new branch. |
| Push (PLUGIN repo) | `cd <plugin-source-repo> && plugin-publish` | One step that pushes, syncs the plugin cache, and bumps the version. Always use this for any repo with `.claude-plugin/plugin.json`. A plain `git push` on a plugin repo is not allowed. |

### RAG CLI

**RAG queries are ALWAYS written in English, regardless of conversation language.**

**`delete` removes everything the scope covers.**
- It removes the matched chunks and their rows in the `indexed_files` manifest.
- It also removes the on-disk source files under `data/documents/<collection>/`.

**`index` is the inverse of `delete` over the same scope.**
- It chunks, embeds, and stores `.md` files from `data/documents/<collection>/`.
- Unchanged files are skipped by default, detected via content hash.

**`search` finds the hit and `read_document` pulls the context around it.**
- `read_document <coll> <doc> <chunk> --before N --after M` returns the chunk plus its neighbors.
   - The useful detail usually sits in those neighbors.

**Miss handling.**
- On a result with zero chunks, reformulate the query at least twice.
   - After two misses, stop and report to the user.
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

**Open issues are the default listing.**
- `gh-cli list_issues` shows open issues by default.

#### What an Issue IS

**An issue is a lean entry point into a topic.**
- The body names the topic and where its content lives, because the content itself lives elsewhere.

**Issues are created at exactly two points.**
- The first point is when the user asks mid-session.
- The second point is Recap, for whatever is still open at session end.

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

**The body carries the GOAL, meaning the end state, and nothing else.**
- The end state is what makes closing the issue a yes-or-no question.
- It is one bullet where the state is single, and several bullets where it genuinely has several parts.

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

| Operation | Command |
|---|---|
| Open one file | `show <path>` |
| Open multiple files | `show <p1> <p2> ...` |
| Relative path | `show ./report.md` |
| Home path | `show ~/Desktop/foo.png` |
