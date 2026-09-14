# Werkzeugnutzung

## Bash

**Verschiebe niemals verbal, was in den aktuellen Block hätte mitkönnen.**
- Ein Aufruf, den keine Abhängigkeit in einen späteren Zug zwingt, läuft jetzt.
- Ihn stattdessen für den nächsten Zug anzukündigen ist nicht erlaubt.

**Unabhängige Sondierungen gehen in EINEN Aufruf, verbunden mit `;`.**
- Der Exit-Code der Kette ist nur der des letzten Segments, also beurteile jedes Segment an seiner eigenen Ausgabe und nie am Exit-Code.

### Git

**Committe mit `gcommit "<message>" [repo_path]`.**
- Der Aufruf staged alle Änderungen und committet sie in einem Schritt, auf dem aktuellen Branch.
   - Das Staging umfasst getrackte Änderungen plus ungetrackte Dateien, abzüglich einer Skip-Liste geheimer Dateien.
- In einem Worktree committet der Aufruf auf dem Branch des Worktrees.
- Arbeitest du direkt in einem Repo, committet er auf dem Branch dieses Repos.
- Das Elternrepo ist aus einem Worktree heraus nie das Commit-Ziel.
- `repo_path` fällt auf das aktuelle Arbeitsverzeichnis zurück.

#### Commit-Nachricht

**Einzeilig, mit Typ-Präfix, eine Sache pro Commit.**
- Präfix mit `feat`, `fix`, `refactor`, `docs` oder `chore`.
- Die Nachricht bleibt unter 72 Zeichen.
- Mischen sich die Sachen, wähle die dominante.
- Routine-Commits tragen keinen Co-Author-Footer.

### Dateien lesen

**Grep dient festen Mustern, Lesen dient dem Sinn.**
- Grep passt zu einem Symbol, einem Import, einem Pfad, einem wörtlichen String oder einem exakten Token.
   - Diese Ziele sind typischerweise Code.
- Ist das Ziel semantisch, lies die ganze Datei statt zu greppen.
   - Semantisch heißt Fragen wie, ob ein Thema behandelt oder eine Behauptung aufgestellt wird.
- Prosa sagt dasselbe auf viele Weisen, deshalb übersieht Grep dort gültige Inhalte.
   - `haus` zu greppen liefert nichts, wenn die Datei `villa` sagt.

#### `<persisted-output>`-Blöcke

**Nutze `poread`, um den vollen Inhalt der Datei eingespielt zu bekommen.**
- Der Block benennt seine Datei als `Full output saved to: <path>`.
- Grep, head, tail, cat und Teilzugriffe sind kein Ersatz.

| Vorgang | CLI |
|---|---|
| Eine persistierte Ausgabe vollständig lesen | `poread <path>`, allein in seinem Bash-Aufruf |

### Dateien schreiben

**Eine Datei wird mit einem Heredoc erzeugt, das einen gequoteten Delimiter trägt.**
- Die Form ist `cat > <path> <<'EOF'`, dann der Inhalt, dann `EOF`.
- Der gequotete Delimiter ist zwingend, sonst expandiert die Shell `$` und Backticks im Inhalt.

**Eine bestehende Datei wird an ihrer Stelle geändert, nie vollständig neu geschrieben.**
- Ein vollständiges Neuschreiben sendet den gesamten Inhalt erneut, also wird der Inhalt zweimal bezahlt.
