# Werkzeugnutzung

## Bash

**Ein Bash-Aufruf umfasst alles, was dein aktueller Denkschritt erfordert.**
- Manifestiere in einem Bash-Call deinen aktuellen Gedankenstand.

**Mehrere Bash-Aufrufe sind die Norm, sofern sie aufeinander aufbauen.**
- Frage dich immer, kann ich das auch parallel ausführen?
   - Falls ja, chaine es in einen Bash mit `;`.
   - Falls nein und du würdest gerne dazwischen denken, ist das vollkommen legitim, dann zwei Bash sequentiell aufeinander aufbauend.

**Der Exit-Code einer mit `;` verbundenen Kette ist nur der des zuletzt ausgeführten Segments.**
- Beurteile deshalb jedes Segment an seiner eigenen Ausgabe und nie am Exit-Code.

### Commit

**Ein Commit staged und committet in einem einzigen Bash-Aufruf.**
- `git add -A` staged alles, was nicht in der `.gitignore` steht.
- Der Pfad hinter `-C` ist immer das Repo, in das committet wird, committet wird auf dessen aktuellen Branch.

**Die Commit-Nachricht ist einzeilig, trägt ein Typ-Präfix und deckt genau eine Sache ab.**
- Das Präfix ist `feat`, `fix`, `refactor`, `docs` oder `chore`.
    - Das folgt Conventional Commits (Conventional Commits 1.0.0, 2019).
- Die Nachricht bleibt unter 72 Zeichen.
- Routine-Commits tragen keinen Co-Author-Footer.

| Vorgang | Command |
|---|---|
| Alle Änderungen stagen und committen | `git -C <repo_path> add -A && git -C <repo_path> commit -m "<message>"` |

### poread

**Nutze `poread`, um den vollen Inhalt der Datei injected zu bekommen.**
- Persisted Output wird erzeugt, wenn eine Ausgabe sehr groß ist.
    - Bekommst du also eine persisted-output Nachricht, ist das für dich ein Zeichen noch einmal darüber nachzudenken, ob du den vollen Output haben möchtest.
    - In vielen Fällen musst du den vollen Output lesen, weil ansonsten der Context verloren geht.
    - Zum Lesen des vollen Outputs nutzt du dann poread.
- Vermeide es, einen vollen Output durch Teiloperationen wie grep, head oder tail zu ersetzen.
    - Lies den Output komplett oder gar nicht.
- Bei Outputs über 50 KB bist du allerdings gezwungen Teiloperationen durchzuführen, hier funktioniert poread nicht mehr.

| Vorgang | Command |
|---|---|
| Eine persistierte Ausgabe vollständig lesen | `poread <path>` |

## Read, Write, Edit

**Nutze `Read` zum Lesen, `Write` zum kompletten Neuerstellen, `Edit` zum Bearbeiten bestehender Dateien.**
- Vermeide Bash bei der Arbeit mit persistenten Files.
