# Tool-Use

## Bash

**Never verbally defer what could have chained into the current block.**
- A call that no dependency forces into a later turn runs now.
- Announcing it for the next turn instead is not allowed.

**Independent probes go into ONE call, joined with `;`.**
- The chain's exit code is only the last segment's, so judge each segment by its own output and never by the exit code.

### Git

**Commit with `gcommit "<message>" [repo_path]`.**
- The call stages all changes and commits them in one step, on the current branch.
   - Staging covers tracked modifications plus untracked files, minus a skip-list of secret files.
- In a worktree the call commits on the worktree's branch.
- Working directly in a repo, it commits on that repo's branch.
- The parent repo is never the commit target from a worktree.
- `repo_path` defaults to the current working directory.

#### Commit Message

**Single-line, type-prefixed, one concern per commit.**
- Prefix with `feat`, `fix`, `refactor`, `docs`, or `chore`.
- The message stays under 72 characters.
- If concerns mix, pick the dominant one.
- Routine commits carry no Co-Author footer.

### Reading files

**Grep serves fixed patterns and reading serves meaning.**
- Grep fits a symbol, an import, a path, a literal string, or an exact token.
   - Those targets are typically code.
- When the target is semantic, read the whole file instead of grepping.
   - Semantic means questions like whether a topic is covered or a claim is made.
- Prose says the same thing many ways, so grep misses valid content there.
   - Grepping `haus` returns nothing when the file says `villa`.

#### `<persisted-output>` blocks

**Use `poread` to get the full content of the file injected.**
- The block names its file as `Full output saved to: <path>`.
- Grep, head, tail, cat, and partial reads are not substitutes.

| Operation | CLI |
|---|---|
| Read a persisted output in full | `poread <path>`, alone in its Bash call |

### Writing files

**A file is created with a heredoc carrying a quoted delimiter.**
- The form is `cat > <path> <<'EOF'`, then the content, then `EOF`.
- The quoted delimiter is mandatory, otherwise the shell expands `$` and backticks inside the content.

**An existing file is changed in place, never rewritten in full.**
- A full rewrite resends the entire content, so the content is paid for twice.
