# Werkzeugnutzung — nur Orchestrator

## worker-cli

**worker-cli findet Projekt, Repos und Modell eines Workers immer selbst.**
- Kein Command nimmt einen Projektpfad oder ein Modell an.
- Der einzige Pfad ist `<target_repo>` bei `worktree`.
- Ein überzähliges Argument bricht ab und nennt die richtige Form.

### Commands

| Vorgang | Command |
|---|---|
| Aktive Worker listen | `worker-cli list` |
| Worker-Status prüfen | `worker-cli status <name>` |
| Ausgabe seit dem letzten Prompt lesen | `worker-cli capture <name> [--raw]` |
| Die letzten N Assistant Turns lesen | `worker-cli response <name> [N]` |
| Nachricht an einen laufenden Worker senden | `worker-cli send <name> <message>` |
| Worker-Branches in allen seinen Repos mergen | `worker-cli merge <name>` |
| Worker killen, samt allen seinen Worktrees und Branches | `worker-cli kill <name>` |
| Worker im Worktree des aktuellen Projekts spawnen | `worker-cli spawn <name> <prompt_file>` |
| Worktree unter `<target_repo>/.claude/worktrees/<name>` erzeugen, auf Branch `<name>` | `worker-cli worktree <name> <target_repo> [branch]` |
| Toten Worker wiederbeleben | `worker-cli revive <name>` |
| Warten, bis die Worker des aktuellen Projekts fertig sind | `worker-cli wait [--timeout SEC]` |

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
