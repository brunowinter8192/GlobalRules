# Tool-Use

## Bash

**Never verbally defer what could have chained into the current block.**
- A call that no dependency forces into a later turn runs now.
- Announcing it for the next turn instead is not allowed.

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

## Read

**Reads a file in `cat -n` format.**
- Each output line shows the line number, a tab, and the content.
- Use `offset` plus `limit` for larger files.
- The tool also reads PNG and JPG images and `.ipynb` notebooks.

### Grep for patterns, Read for meaning

**Grep serves fixed patterns and Read serves meaning.**
- Grep fits a symbol, an import, a path, a literal string, or an exact token.
   - Those targets are typically code.
- When the target is semantic, read the file instead of grepping.
   - Semantic means questions like whether a topic is covered or a claim is made.
- Prose says the same thing many ways, so grep misses valid content there.
   - Grepping `haus` returns nothing when the file says `villa`.

### `<persisted-output>` blocks

**A `<persisted-output>` block is always Read in full.**
- The block names its file as `Full output saved to: <path>`.
- Extract that absolute path and Read the entire file in one call.
- If the file exceeds 2000 lines, raise `limit=N` to cover it.
- Grep, head, tail, and partial reads via offset are not acceptable substitutes.

## Edit

**Performs exact string replacement in a file.**
- Read the file in this conversation before editing, or the call fails.
   - One Read per file per session suffices, because the file counts as read for later Edits.
- `old_string` must match the file exactly, including indentation, and must be unique.
   - Strip the Read line prefix of number and tab before matching.
- With `replace_all` set to true, every occurrence is replaced instead.

## Write

**Writes a file to the local filesystem, overwriting if one exists.**
- Use it to create a new file or fully replace one you already Read.
- Overwriting an unread file fails.
- For partial changes, use Edit instead.
- Prefer Edit for existing files, because Write resends the full content every time.
