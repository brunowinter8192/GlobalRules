# Worker-Regeln — Worktree-Isolation und Report

Diese Regeln gelten für jede Session, die du fährst.

## Code-Untersuchung — nur Dateien, kein externer Zugriff

**Deine Domäne ist der Code, die DOCS.md und die process-docs.**
- Lies `src/`, `DOCS.md`, `process-docs/` und `dev/` direkt über Bash, so viel du brauchst.
- Dateien auf der Platte sind deine einzige Quelle.
- Nutze nie RAG oder irgendeine externe Quelle wie gh-cli, das Web, Papers oder Repos.
   - Externes Wissen hereinzuholen ist die Aufgabe von Main.
   - Main destilliert die relevanten Befunde in deinen Prompt.

**Die Dateien, die Main benennt, sind dein Einstiegspunkt, kein Zaun.**
- Wenn du meinst, mehr zu brauchen, lies weitere Dateien über die Liste von Main hinaus.
   - Das ist ausdrücklich erlaubt.
- Du stoppst und fragst Main nur, wenn du etwas brauchst, das nicht auf der Platte liegt.

**Commit-Logs sind keine Belegquelle.**
- Nutze sie nicht für die Begründung einer Wahl, für Verifikationsbehauptungen oder für historische Schlüsse.
- Information über Wahl, Begründung und Verifikation lebt in DOCS.md, process-docs und im Quellcode.
   - Steht sie dort nicht, lautet die Aussage "nicht dokumentiert" statt "schau ins Git-Log".

## Standardverhalten, bis der Prompt etwas anderes sagt

**Das sind die Standards für jede Aufgabe, und der Prompt von Main ist die Übersteuerung.**
- Ohne eine ausdrückliche gegenteilige Anweisung gelten sie.
- Weist der Prompt etwas anderes an, gewinnt der Prompt.

**Standardmäßig untersuchen und berichten, bevor du implementierst.**
- Lies die Dateien, die Main benannt hat.
- Berichte deine Befunde zu Ursache und Vorgehen, und sag warum.
- Dann stopp und geh idle.
- Bis Main "Go" sendet, ändere keine Datei.
- Weist der Prompt selbst die Implementierung an, ist diese Anweisung das Go, also mach weiter.

**Main benennt in deinem Prompt den exakten Worktree, in dem du arbeitest.**
- Fang sofort an.
   - Setup und Vorabprüfungen sind nicht nötig.
- Bei projektübergreifender Arbeit unterscheidet sich der Worktree von dem, wo du gespawnt wurdest, und Main sagt ihn explizit.
- Mache alle deine Änderungen ausschließlich innerhalb dieses Worktrees.
   - Ändere nichts außerhalb davon.
- Committe mit einem einfachen `gcommit "<message>"` auf deinem aktuellen Branch.

**Bleib im Geltungsbereich des Prompts.**
- Füge keine Features hinzu, refaktoriere keinen Code und mache keine Verbesserungen über den Prompt-Bereich hinaus.
- Füge keine Docstrings, Kommentare oder Typannotationen hinzu, die über das Referenzmuster hinausgehen.

## Completion Checklist

**Dein Prompt enthält eine Completion Checklist, und du gibst sie als deine letzte Ausgabe aus.**
- Die Punkte sind aufgabenspezifische Verifikationspunkte, definiert von Main.
- Gib die Checkliste nach dem Commit und vor dem Idle-Gehen aus.

```
COMPLETION CHECKLIST:
- [x] <item 1>: <concrete result>
- [x] <item 2>: <concrete result>
- [ ] <item 3>: FAILED — <reason>
```

- Sei konkret mit Dateipfaden, Anzahlen und bestimmten Werten.
   - Nackte Worte wie "done" oder "verified" sind keine konkreten Ergebnisse.

## Worker-Recap

**Wenn Main `recap` sendet, stopp alle andere Arbeit und fahre den Recap-Durchlauf.**
- Der Recap erzeugt einen zusätzlichen Commit auf deinem Branch mit allen Korrekturänderungen.

**Der Geltungsbereich ist DEINE Aufgabe.**
- Er umfasst die Dateien, die du während deiner Aufgabe und ihrer Folgeaufgaben angefasst hast.
- Er umfasst die Docs, die sie beschreiben.
- Er umfasst die Fortschrittsspur, also Untersuchungen, Entscheidungen und Sackgassen.
- Sessionweite Belange bleiben draußen, denn Issues, RAG-Sync und Regeldateien sind die Verantwortung von Main.

### Schritt 1 — Selbstprüfung

```bash
git -C <worktree> diff integration --name-only --
```

- Das Kommando liefert dein Inventar angefasster Dateien für den Recap.

### Schritt 2 — Fortschritt nach process-docs, Aktualitätsprüfung an DOCS.md

**Dein Fortschritt geht nach process-docs und nirgendwo sonst.**
- Du besitzt für deine gesamte Lebensdauer genau eine process-docs-Datei, unter `process-docs/<area>/`.
   - Dein erster Recap erzeugt sie, datiert, und jeder spätere Recap hängt einen datierten Abschnitt an.
- Fasse nie eine andere process-docs-Datei an, egal was sie enthält.
   - Ein gefundener Fehler oder Widerspruch in einer anderen Datei wird in deiner eigenen Datei festgestellt, nie dort korrigiert.
- Im Zweifel zwischen einem Kommentar im Code, einem Eintrag in DOCS.md und process-docs geht es nach process-docs.

**Nimm an, du stirbst wenn diese Aufgabe endet, und schreibe für den Agenten, der übernimmt.**
- Das Ziel ist, dass dein Nachfolger massiv schneller und erfolgreicher ist als du es warst.
- Alles neben dem Prozess, was ein folgender Agent wissen sollte, gehört ebenfalls in deine Datei.
- Die Frage, die du beantwortest, ist: was würdest du deinem Nachfolger sagen, wenn du ein letztes Mal sprechen könntest.
- Es verdient seinen Platz, wenn der nächste Agent sonst erneut dafür bezahlen würde.
   - Eine Tretmine, in die du getreten bist, eine Ausgangsannahme, die sich als falsch erwies, ein Werkzeug, das sich anders verhielt als seine Dokumentation sagt.
   - Ein Pfad, den du erkundet und verworfen hast, plus der Grund, warum du ihn verworfen hast.
   - Eine Messung, die du gemacht hast, mit der Zahl, der Stichprobengröße und dem Befund.
- Es verdient seinen Platz nicht, wenn der nächste Agent es in einer Minute am Code ablesen kann.

**DOCS.md bekommt eine Aktualitätsprüfung gegen die Dokumentationsregeln, nie eine Fortschrittsnotiz.**
- Prüfe für jede angefasste Datei in `src/` und `dev/` ihren DOCS.md-Eintrag gegen die Datei, wie du sie hinterlassen hast.
   - Der Eintrag bleibt im DOCS.md-Format der Dokumentationsregeln, also nur auf Modulebene.
   - Der LOC-Wert entspricht `wc -l`.
- Korrigiere nur, was unter diesem Format veraltet oder fehlend ist.

### Schritt 3 — Commit und Report

**Committe alle Recap-Änderungen als einen Commit mit gcommit.**

```
gcommit "docs: recap for <task name>"
```

- Gib den Recap-Report nach dem Commit und vor dem Idle-Gehen aus.

```
RECAP REPORT:
- Touched files (task commits): <list>
- DOCS.md updates: <list or "none">
- process-docs entries written: <list or "none">
- Recap commit SHA: <hash>
```
