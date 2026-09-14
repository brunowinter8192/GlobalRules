# Werkzeugnutzung — nur Orchestrator

## Bash

### Worker CLI

**Worker-Namen sind global eindeutig.**
- Ein Register führt jeden Worker-Namen.
- Nur `spawn` verlangt den Projektpfad.
- Bei einem projektübergreifenden Worker hänge `<project_path>` explizit an jedes spätere Kommando.

**`worker-cli response` ist der Standard, um idle Worker zu lesen.**
- `response` liefert sauberen Assistententext aus dem Session-JSONL.
- `capture` ist der Leser, wenn `status` `dead` zeigt.

**Muster des Session-Namens.**
- Das Muster ist `worker-<basename(project_path)>-<name>`.

| Vorgang | CLI |
|---|---|
| Aktive Worker listen (Projekt) | `worker-cli list <project_path>` |
| Aktive Worker listen (alle) | `worker-cli list` |
| Worker-Status prüfen | `worker-cli status <name> [project_path]` |
| Saubere Ausgabe seit dem letzten Prompt | `worker-cli capture <name> [project_path]`. Mit `--raw` schreibt es das rohe Pane in eine Datei. |
| Saubere letzte N Assistentenzüge (JSONL) | `worker-cli response <name> [N] [project_path]` |
| Nachricht an laufenden Worker senden | `worker-cli send <name> <message>` |
| Worker-Branch mergen | `worker-cli merge <name> [project_path]` |
| Worker killen (+ registrierte projektübergreifende Worktrees) | `worker-cli kill <name> [project_path]` |
| Worker im Worktree spawnen | `worker-cli spawn <name> <prompt_file> <project_path> [model] [--no-worktree]` |
| Projektübergreifenden Worktree erzeugen | `worker-cli worktree <name> <target_repo> [branch]` |
| Toten Worker wiederbeleben (CC-Session fortsetzen) | `worker-cli revive <name>` |

### Git

| Vorgang | CLI | Hinweise |
|---|---|---|
| Push (NICHT-Plugin-Repo) | `git -C <repo_path> push` | Fällt auf `-u origin <branch>` zurück, wenn kein Upstream existiert. Nutze stattdessen `plugin-publish`, wenn `.claude-plugin/plugin.json` existiert. |
| Push mit Upstream (NICHT-Plugin-Repo) | `git -C <repo_path> push -u origin $(git -C <repo_path> branch --show-current)` | Für den ersten Push auf einem neuen Branch. |
| Push (PLUGIN-Repo) | `cd <plugin-source-repo> && plugin-publish` | Ein Schritt, der pusht, den Plugin-Cache synchronisiert und die Version hochzieht. Nutze das immer für jedes Repo mit `.claude-plugin/plugin.json`. Ein einfaches `git push` auf einem Plugin-Repo ist nicht erlaubt. |

### RAG CLI

**RAG-Abfragen werden IMMER in Englisch geschrieben, unabhängig von der Gesprächssprache.**

**`delete` entfernt alles, was der Geltungsbereich abdeckt.**
- Es entfernt die getroffenen Chunks und ihre Zeilen im `indexed_files`-Manifest.
- Es entfernt außerdem die Quelldateien auf der Platte unter `data/documents/<collection>/`.

**`index` ist die Umkehrung von `delete` über denselben Geltungsbereich.**
- Es zerlegt, embeddet und speichert `.md`-Dateien aus `data/documents/<collection>/`.
- Unveränderte Dateien werden standardmäßig übersprungen, erkannt über den Inhalts-Hash.

**`search` findet den Treffer und `read_document` holt den Kontext darum.**
- `read_document <coll> <doc> <chunk> --before N --after M` liefert den Chunk plus seine Nachbarn.
   - Das nützliche Detail sitzt meist in diesen Nachbarn.

**Umgang mit Fehlschlägen.**
- Bei einem Ergebnis mit null Chunks formuliere die Abfrage mindestens zweimal um.
   - Nach zwei Fehlschlägen stopp und melde dem Nutzer.
- Bei einem Teiltreffer führe `read_document` um den Chunk-Index des Treffers aus, statt neu abzufragen.

| Vorgang | Kommando |
|---|---|
| Sammlungen listen | `rag-cli list_collections [--filter PATTERN]` |
| Dokumente listen | `rag-cli list_documents <collection> [--document PATTERN] [--exclude PATTERN] [--filter PATTERN]` |
| Suchen | `rag-cli search <query> <collection> [--document PATTERN] [--exclude PATTERN]` |
| Kontext lesen | `rag-cli read_document <collection> <doc.md> <chunk> [--before N] [--after N]` |
| Löschen | `rag-cli delete --collection <name> [--document <doc>]` |
| Indexieren | `rag-cli index --collection <name> [--document <doc>]` |

### GitHub Issues (gh-cli) — sessionübergreifender Kontext

**Leite `<owner>` und `<repo>` vom Git-Remote ab.**
- `git remote get-url origin` liefert `github.com:<owner>/<repo>.git`.

**Offene Issues sind die Standardauflistung.**
- `gh-cli list_issues` zeigt standardmäßig offene Issues.

#### Was ein Issue IST

**Ein Issue ist ein schlanker Einstiegspunkt in ein Thema.**
- Der Body benennt das Thema und wo sein Inhalt liegt, denn der Inhalt selbst liegt anderswo.

**Issues werden an genau zwei Punkten erstellt.**
- Der erste Punkt ist, wenn der Nutzer mitten in der Session danach fragt.
- Der zweite Punkt ist der Recap, für alles, was am Sessionende noch offen ist.

#### Issue-Format

```
<Title: ONE word>

Goal:
- <end state>
- <end state>

Area: <area>  (→ process-docs/<area>/, dev/<area>/)
```

**Der Titel ist EIN Wort, und er benennt die Sache, nie die Handlung.**
- `Wohnungsmängel`, `Bügeleisen` und `Hausarzt` sind Titel, `Zahnarzt in Frankfurt finden und Kontrolle 2026` ist keiner.

**Der Body trägt das ZIEL, also den Endzustand, und nichts anderes.**
- Der Endzustand ist das, was das Schließen des Issues zu einer Ja-Nein-Frage macht.
- Er ist ein Bullet, wo der Zustand einzeln ist, und mehrere Bullets, wo er wirklich mehrere Teile hat.

| Vorgang | CLI |
|---|---|
| Offene Issues listen | `gh-cli list_issues <owner> <repo>`. Offen ist der Standardzustand. |
| Geschlossene Issues listen | `gh-cli list_issues <owner> <repo> --state closed` |
| Issue-Body lesen | `gh-cli get_issue <owner> <repo> <number>`. Der Body ist der Text nach dem `---`-Trenner in der Ausgabe. |
| Issue erstellen | `gh-cli create_issue <owner> <repo> "<title>" --body "<desc>" [--labels a,b]` |
| Issue-Body aktualisieren (nur Bereichswechsel) | `gh-cli update_issue <owner> <repo> <number> --body "<full updated body>"`. Der Aufruf ersetzt den gesamten Body. |
| Issue schließen | `gh-cli update_issue <owner> <repo> <number> --state closed` |
| Issue wieder öffnen | `gh-cli update_issue <owner> <repo> <number> --state open` |

### show — eine Datei für den Nutzer öffnen

**Öffne eine Datei in der Standard-macOS-App des Nutzers, damit der NUTZER sie sehen kann.**
- Nutze es, wenn der Nutzer darum bittet, eine Datei gezeigt zu bekommen, etwa "öffne mir den Report" oder "show me X".
- Der Auslöser ist die Absicht des Nutzers, sie anzusehen, und nie der Dateityp.

**Nutze `show` nur, wenn der Nutzer eine Datei ANSEHEN will.**
- Für deine eigene Inspektion wie Analyse, Code-Review oder Grep nutze Bash.

**Eine schon mit `show` geöffnete Datei bleibt offen.**
- Ein `show` bei der ersten Anzeige gilt für die ganze Session.

| Vorgang | Kommando |
|---|---|
| Eine Datei öffnen | `show <path>` |
| Mehrere Dateien öffnen | `show <p1> <p2> ...` |
| Relativer Pfad | `show ./report.md` |
| Home-Pfad | `show ~/Desktop/foo.png` |
