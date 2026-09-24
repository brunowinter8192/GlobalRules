# Werkzeugnutzung — nur Orchestrator

## worker-cli

**Bei einem projektübergreifenden Worker hängst du `project_path` an jedes Command.**
- `merge`, `kill`, `status`, `capture` und `response` nehmen es als letztes Argument.
- Ohne die Angabe lösen sie auf das Projekt auf, in dem der Worker gespawnt wurde.

### Commands

| Vorgang | Command |
|---|---|
| Aktive Worker listen | `worker-cli list [project_path]` |
| Worker-Status prüfen | `worker-cli status <name> [project_path]` |
| Ausgabe seit dem letzten Prompt lesen | `worker-cli capture <name> [project_path] [--raw]` |
| Die letzten N Assistant Turns lesen | `worker-cli response <name> [N] [project_path]` |
| Nachricht an einen laufenden Worker senden | `worker-cli send <name> <message>` |
| Worker-Branch mergen | `worker-cli merge <name> [project_path]` |
| Worker killen, samt allen seinen Worktrees und dem Branch | `worker-cli kill <name> [project_path]` |
| Worker im Worktree spawnen | `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` |
| Worktree unter `<target_repo>/.claude/worktrees/<name>` erzeugen, auf Branch `<name>` | `worker-cli worktree <name> <target_repo> [branch]` |
| Toten Worker wiederbeleben | `worker-cli revive <name>` |
| Warten, bis die Worker des Projekts fertig sind | `worker-cli wait [project_path] [--timeout SEC]` |

### capture

#### Input args

- `--raw` — schreibt statt der sauberen Ausgabe das rohe Pane in eine Datei.

### wait

#### Input args

- `--timeout SEC` — der Standard sind 3300 Sekunden.

## git

**Ein Repo mit `.claude-plugin/plugin.json` ist ein Plugin-Repo und wird nur über `plugin-publish` gepusht.**
- `plugin-publish` pusht, synchronisiert den Plugin-Cache und zieht die Version hoch, alles in einem Schritt.
- Ein einfaches `git push` auf einem Plugin-Repo ist nicht erlaubt.

### Commands

| Vorgang | Command |
|---|---|
| Push in einem Nicht-Plugin-Repo | `git -C <repo_path> push` |
| Erster Push eines neuen Branches | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` |
| Push in einem Plugin-Repo | `cd <plugin-source-repo> && plugin-publish` |

## rag-cli

**RAG-Queries werden IMMER in Englisch geschrieben, unabhängig von der Gesprächssprache.**

### Commands

| Vorgang | Command |
|---|---|
| Collections listen | `rag-cli list_collections [--filter PATTERN]` |
| Dokumente listen | `rag-cli list_documents <collection> [--document PATTERN] [--exclude PATTERN] [--filter PATTERN]` |
| Suchen | `rag-cli search <query> <collection> [--document PATTERN] [--exclude PATTERN]` |
| Chunks erweitern | `rag-cli expand_chunks <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Löschen | `rag-cli delete --collection <name> [--document <doc>]` |
| Indexieren | `rag-cli index --collection <name> [--document <doc>]` |

### search

**Bei einem Ergebnis mit null Chunks formulierst du die Query mindestens zweimal um.**
- Nach zwei Fehlschlägen stoppst du und meldest es dem User.
- Bei einem Teiltreffer nutzt du `expand_chunks` um den Chunk-Index des Treffers, statt neu zu suchen.

#### Input args

- `--document PATTERN` — begrenzt die Suche auf Dokumente, die auf das Muster passen.
- `--exclude PATTERN` — nimmt Dokumente aus, die auf das Muster passen.

### expand_chunks

**`search` findet den Treffer, `expand_chunks` holt den Kontext darum.**
- `expand_chunks` liefert immer nur die angeforderten Nachbarchunks, nie das ganze Dokument.
- Der Weg zu mehr Kontext aus einer indexierten Datei ist immer `expand_chunks`, nie das `Read`-Tool auf die Quelldatei.

#### Input args

- `<chunk>` — der Chunk-Index aus dem Treffer von `search`.
- `--before N` und `--after N` — nehmen N Nachbarn vor und hinter dem Chunk dazu, dort sitzt meist das nützliche Detail.

### delete

**`delete` entfernt die getroffenen Chunks, ihre Zeilen im `indexed_files`-Manifest und die Quelldateien auf der Platte.**
- Die Quelldateien liegen unter `data/documents/<collection>/`.

#### Input args

- `--document <doc>` — begrenzt den Scope auf ein Dokument, ohne die Angabe trifft es die ganze Collection.

### index

**`index` ist die Umkehrung von `delete` über denselben Scope.**
- Es zerlegt, embeddet und speichert `.md`-Dateien aus `data/documents/<collection>/`.
- Unveränderte Dateien werden standardmäßig übersprungen, erkannt über den Inhalts-Hash.

#### Input args

- `--document <doc>` — begrenzt den Scope auf ein Dokument, ohne die Angabe trifft es die ganze Collection.

## gh-cli

**Leite `<owner>` und `<repo>` vom Git-Remote ab.**
- `git remote get-url origin` liefert `github.com:<owner>/<repo>.git`.

### Commands

| Vorgang | Command |
|---|---|
| Issues listen | `gh-cli list_issues <owner> <repo> [--state open\|closed]` |
| Issue-Body lesen | `gh-cli get_issue <owner> <repo> <number>` |
| Issue erstellen | `gh-cli create_issue <owner> <repo> "<title>" --body "<desc>" [--labels a,b]` |
| Issue-Body ersetzen | `gh-cli update_issue <owner> <repo> <number> --body "<full updated body>"` |
| Issue schließen oder wieder öffnen | `gh-cli update_issue <owner> <repo> <number> --state closed\|open` |

### list_issues

#### Input args

- `--state` — `open` oder `closed`. Ohne die Angabe siehst du nur die offenen Issues.

### update_issue

#### Input args

- `--body` — ersetzt den gesamten Body und wird nur für einen Bereichswechsel genutzt.
- `--state` — `closed` oder `open`.

## show

**Nutze den Command nur wenn der User darum bittet eine Datei gezeigt zu bekommen.**

**Eine schon mit `show` geöffnete Datei bleibt dauerhaft offen.**
- Öffne eine Datei wirklich nur wenn der User im Einzelfall darum bittet.

### Commands

| Vorgang | Command |
|---|---|
| Eine oder mehrere Dateien öffnen | `show <path> [<path> ...]` |

#### Input args

- `<path>` — darf relativ sein oder mit der Tilde beginnen, also `./report.md` oder `~/Desktop/foo.png`.
