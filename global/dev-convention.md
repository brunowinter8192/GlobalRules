# dev/ Directory Convention

## Output Layout

**A dev script writes its report into `dev/<area>/`.**
- Writing to the console instead is not allowed.
- Inside `dev/<area>/` the report goes to `md/`, `csv/`, or `png/`, chosen by output type.
- The report file carries a descriptive name that traces to its producing script.

**Data outputs stay separate from reports.**
- Scripts also produce data outputs like raw corpora.
   - Data outputs go into their own type-named folder, for example `jsonl/`.
- Data folders never mix into `md/`.

**Reports and data organize by theme in `dev/<area>/`.**
- A dev area and a process-docs area on the same theme share one name.

## Staging

**One-shot scripts live in the worktree or /tmp/ and are never staged.**
- Build forensics and one-shot assertions in the worktree or under /tmp/.
   - Explicitly do not stage them on merge.
- A one-shot assertion that becomes a regression guard folds into an existing dev/ test file.
   - A new file per fix is not allowed.
