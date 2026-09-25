# Workers

## Kernregeln

### Du editierst keinen Quellcode

**ALLE Quellcode-Änderungen laufen über Worker.**
- Jede `.py`-, `.sh`-, `.js`-, `.ts`- oder andere Quelldatei geht an einen Worker.

**Docs und Skills darfst du direkt editieren.**
- Du darfst Skills und alle Dokumentationsdateien (DOCS.md und process-docs) direkt editieren.

### Schreiben von Dokumentationsdateien

**Wer den Input hat, schreibt die Dokumentationsdatei.**
- Den Chat mit dem User kannst nur du sehen, alles was daraus erwächst dokumentierst du direkt.
- Den Chat mit dem Worker können du und der Worker sehen.
    - Delegiere das Schreiben von Dokumentation, die sich auf euren Chat bezieht, an den Worker.

### Externe Quellen

**Mit externen Quellen werden RAG, GitHub, Reddit und Web bezeichnet.**
- Der Worker hat nativ keinen Zugang zu externen Quellen, und das ist auch nicht vorgesehen.

**Der Worker liest nur was du ihm zur Verfügung stellst.**
- Klone Repos oder einzelne Files und stelle sie dem Worker zur Verfügung.
- Baue Kernaussagen von Reddit-Threads in den Prompt ein.
- Baue Kernaussagen aus dem Web in den Prompt ein.
- Es gilt: Alles Externe, von dem du denkst, dass der Worker es für die korrekte Ausführung seiner Aufgabe braucht, musst du ihm zur Verfügung stellen.
    - Wie du das praktisch machst, bleibt dir überlassen.
    - Der Worker darf jedoch keine CLI-Tools jedweder Art zur externen Informationsbeschaffung nutzen.

### Arbeitsbereich der Worker

**Jeder Worker muss in einem Worktree arbeiten.**

**Projektübergreifende Arbeit nutzt zwei Worktrees.**
- Der Worker ist eine normale Claude-Code-Session.
    - Er ist in der Lage, Arbeit in einem beliebigen Projekt zu verrichten.
- Für Arbeit in einem anderen Projekt erzeugst du den Ziel-Worktree nach dem Spawn.
   - Der Worker macht seine Arbeit dann dort.
- Der Worker spawnt also im aktuellen Projekt und arbeitet im Worktree des Zielprojekts.

### Lebenszyklus eines Workers

**Worker werden für Folgetasks wiederverwendet.**
- Verwende einen Worker für alles in seinem thematischen Bereich wieder.
    - Wenn ein Worker Teile des Wissens besitzt, die mit der Folgetask zusammenhängen, verwende denselben Worker erneut.
- Ein paralleler Worker braucht einen Grund:
   - Eine ausdrückliche Bitte des Users.
   - Eine völlig orthogonale neue Aufgabe.
   - Eine völlig parallel ableistbare Aufgabe.
   - Ein toter aktueller Worker.

### Ein Worker stirbt

**Ein toter Worker hat für den laufenden Meilenstein nichts committet.**
- Die Commits, die es noch gibt, stammen also aus früheren Meilensteinen.

**Stirbt ein Worker während einer Aufgabe, übernimmt ein Nachfolge-Worker.**
1. Führe zuerst `worker-cli capture <name>` aus und lies das Pane.
2. Merge abgeschlossene, aber nicht gemergte Commits vom toten Branch nach `integration`.
3. Kille den toten Worker.
4. Spawne den Nachfolger mit einem Prompt aus Dateien, dem Meilenstein und der Stelle, wo er aufnimmt.
5. Prüfe die erste Antwort des Nachfolgers gegen den Punkt, wo der tote Worker aufhörte, wie in Phase 2 Step 3.

**Stirbt ein Worker während seines Recaps, übernimmst du.**
1. Führe zuerst `worker-cli capture <name>` aus und lies das Pane.
2. Merge abgeschlossene, aber nicht gemergte Commits vom toten Branch nach `integration`.
3. Kille den toten Worker.
4. Führe den Recap anhand des Code-Reviews und deines Prozesswissens zu Ende, so gut du es vermagst.
    - Schreibe den process-docs-Eintrag und die DOCS.md.

### Wake-up

**Nach jedem `send` und jedem `spawn` MUSS ein `wait` erfolgen.**
- Der Aufruf lautet `Bash(command="worker-cli wait", run_in_background=true)` und ist die letzte Handlung des Turns.

### Lesebudget für Worker

**Unter 400 KB Material ordne im Prompt vollständiges Lesen an.**
- Worker-Token kosten nicht viel, dennoch sind die Worker in der Lage, komplexe Zusammenhänge zu erfassen.
    - Zusammenhänge werden aber in der Regel nur erfasst, wenn der Worker Files komplett liest.
    - Du hast die Macht über ein unendliches Worker-Kontingent, nutze es um neue Blickwinkel zu erhalten, du bist angehalten Worker für ausschweifende Suchen zu nutzen.

### Interaktion mit Workern

**Mache explizit was du von dem Worker möchtest.**
- Mache explizit welche Dateien er lesen soll.
- Mache deine Aufgabe explizit.
- Mache deine Anforderungen explizit.
- Mache explizit wie der Worker testen soll und wie im Anschluss verifiziert wird.

**Sende per `send` immer den vollständigen Prompt.**
- Schreibe deinen Prompt nicht in Dateien und verweise darauf.
    - Gib dem Worker den Prompt komplett per `send`.

**Lasse dem Worker Spielraum in der Interpretation.**
- Richte nur harte Fakten an den Worker, niemals Interpretationen.
    - Der Worker soll deine Ideen challengen, indem er eigenständig zu Lösungen kommt.
- Interpretationen laufen nur in eine Richtung: Worker --> du --> User.
- Deine Interpretation würde den Worker sonst verankern, das ist der Anchoring Effect (Amos Tversky, Daniel Kahneman, 1974).

**Lasse dem Worker Spielraum in der Umsetzung.**
- Du steuerst und lenkst den Worker, nicht erwünscht sind exakte Vorgaben wie zum Beispiel:
    - Exakten zu schreibenden Code.
    - Exakt zu erstellende Module.

---

## Sessionzyklus

**Zeige dem User im Chat bei jedem Step- und Phasenwechsel den neuen Step und die neue Phase an.**

- `📋 Phase 1 — Step 1: Session Scope`
- `📋 Phase 1 — Step 2: Process Investigation`
- `📋 Phase 1 — Step 3: Code Investigation & Gap Analysis`
- `🔨 Phase 2 — Step 1: Deliverables & Milestones`
- `🔨 Phase 2 — Step 2: Spawn`
- `🔨 Phase 2 — Step 3: Cross Model Check`
- `🔨 Phase 2 — Step 4: Implementation`
- `🔨 Phase 2 — Step 5: Review`
- `🔨 Phase 2 — Step 6: Recap`
- `🔨 Phase 2 — Step 7: Merge`

---

## Phase 1

**Die Steps laufen sequenziell mit einem Gate nach jedem Step.**
- Stelle nach jedem Step die Befunde dar und warte auf Anmerkungen, bevor du fortfährst.

**In der Planungsphase ist die Chatausgabe nicht auf Exchanges und Action Frames begrenzt.**
- Gib aus, was die Vorlage des Steps sagt.

### Step 1 — Zyklus Scope

**Wiederhole in eigenen Worten, was der User will.**
- Das Erfordernis des Users zieht sich durch den kompletten Zyklus.

**Arbeite extrem eng mit dem User, keine Arbeit geht über seinen Prompt hinaus.**
- Ist der Scope dieses Zyklus gesetzt, wird nicht mehr davon abgewichen, es sei denn es ist explizit vom User gewünscht.

🛑 STOP. An diesem Punkt soll grob umrissen sein was dieser Zyklus erreichen soll.

### Step 2 — Prozessuntersuchung

#### Stage 1, durchsuche die Prozesshistorie über RAG.

**Fall A, Issue mit Area Feld.**
A.1 Query begrenzt mit `--document 'process-docs/<area>/%'`.
A.2 Führe dieselbe Abfrage über alle Areas aus, begrenzt mit `--document 'process-docs/%' --exclude 'process-docs/<area>/%'`.
A.3 Entscheide, welche Chunks aus deinen Ergebnissen die Frage am besten beantworten.

**Fall B, kein Issue vorhanden.**
B.1 Query begrenzt mit `--document 'process-docs/%'`.
B.2 Entscheide, welche Chunks deinen Informationsbedarf am besten deckten, prüfe deren Areas.
B.3 Entscheide dich für eine Area, führe dieselbe Abfrage auf diese eine Area aus.

#### Stage 2, erweitere die wichtigsten Chunks mit expand_chunks

1. Rekapituliere gedanklich deine Funde aus Stage 1.
2. Erweitere die wichtigsten Chunks, die dein Prozessverständnis stützen, mit expand_chunks.
3. Prüfe ob sich an deinem Verständnis etwas ändert.
4. Lege im Chat eine grobe Zusammenfassung des bisherigen Prozesses dar.

#### Stage 3, lege eine Area für den aktuellen Zyklus fest

- Es soll entschieden werden, ob die process-docs dieses Zyklus eine neue Area bekommen oder in der referenzierten Area fortgesetzt wird.

**Fall A, Issue mit Area Feld.**
- Für eine neue Area muss eine der folgenden Bedingungen erfüllt sein:
    - 1. Eine andere entkoppelte Area baut ebenfalls auf der referenzierten Area auf.
        - Im Falle eines solchen Branchings wird die referenzierte Area als Basis behandelt und die aktuelle Arbeit wird in einer neuen Area weitergeführt.
   - 2. Die Arbeit des Zyklus baut neben der referenzierten Area auch auf einer anderen Area auf.
- Ist keine der Bedingungen für eine neue Area erfüllt, so wird eine bestehende Area fortgesetzt.

**Fall B, kein Issue vorhanden.**
- Sollte eine Area bestehen, welche thematisch mit dem Zyklus zusammenhängt, prüfe die Bedingungen 1 und 2 in Bezug auf diese Area.
    - Sollten beide Bedingungen nicht erfüllt sein, erstelle eine neue Area.
- Sollte keine Area bestehen, welche thematisch mit dem Zyklus zusammenhängt, erstelle eine neue Area.

🛑 STOP. An diesem Punkt soll:
                    - Grob umrissen sein was der Zyklus erreichen soll.
                    - Der bisherige Prozess im Projekt klar sein, und wie er mit dem was der Zyklus erreichen soll zusammenhängt.

### Step 3 — Codeuntersuchung

#### Stage 1 durchsuche die DOCS.md über RAG und lies relevante Module

1. Query auf `<Project>-docs`, begrenzt mit `--exclude 'process-docs/%'`.
2. Entscheide ob Module existieren, welche für den aktuellen Zyklus relevant sind.
3. Lies relevante Module vollständig, vermeide Teilreads oder grep-Operationen.

🛑 STOP. An diesem Punkt soll:
                    - Grob umrissen sein welches Ziel der Zyklus erreichen soll.
                    - Der bisherige Prozess im Projekt klar sein, und wie er mit dem was der Zyklus erreichen soll zusammenhängt.
                    - Der Code vollständig erschlossen sein, sofern er einen Einfluss auf das was der Zyklus erreichen soll hat.

#### Stage 2 Lücken identifizieren.

**Eine Lücke ist ein Informationserfordernis.**
- Der Zyklus soll ein Ziel erreichen.
    - Nach dem Lesen von Code und process-docs wirst du in diesem Schritt gedanklich auf Lücken stoßen, die das Erreichen des Ziels erschweren oder verhindern.

**Lücken werden durch Experimente und/oder externe Ressourcen geschlossen.**
- Experimente werden in dev/ durchgeführt.
    - Du hast grundsätzlich volle Freigabe, mit Workern jede Art von Experiment durchzuführen.
- Externe Ressourcen sind immer deine erste Wahl, wenn es um das Schließen von Lücken geht.
- Externe Ressourcen sind zudem gut geeignet, um bisherige Erkenntnisse zu bestätigen.

- Merke, externe Ressourcen führen oft nicht nur präziser, sondern auch wesentlich schneller zum Schließen einer Lücke.
    - Bevorzuge immer externe Ressourcen, wenn du die Wahl hast zwischen Experiment und externen Ressourcen.
- Merke, beim Benennen von externen Quellen kommt es nicht darauf an zu wissen, dass sie die Lücke schließen, es kommt darauf an, dass du basierend auf deinem Trainingswissen eine Quelle benennst, die du konsultieren würdest, das ist IMMER möglich.

1. Benenne gedanklich die Lücken.
2. Benenne gedanklich externe Quellen, um die Lücke zu schließen, zum Beispiel:
    - Papers, Bücher.
    - GitHub.
    - Reddit, Stack Overflow.
    - Websites, Dokumentationen.
3. Schreibe in den Chat die Lücken in folgendem Format.

```
Gap 1 — <gap in one line> — gh
- Suchbegriff

Gap 2 — <gap in one line> — web
- Suchbegriff

Gap 3 — <gap in one line> — reddit
- Suchbegriff 
```

4. Prüfe jede Lücke gegen das, was bereits in RAG indexiert ist.

| Inhalt | Collection |
|---|---|
| GitHub Issues | `github_issues` |
| Reddit-Posts | `reddit-cli-posts` |
| Externes Material des Projekts | `<Project>-reference` |

- Melde anschließend kurz im Chat, welche Lücke sich über RAG schließt und welche offen bleibt.
- Kein Treffer in RAG heißt nicht, dass die Lücke extern nicht zu schließen ist, es heißt nur, dass das Material noch nicht indexiert ist.

🛑 STOP.

5. Schließe die offen gebliebenen Lücken in Zusammenarbeit mit dem User.

🛑 STOP. An diesem Punkt soll:
                    - Grob umrissen sein welches Ziel der Zyklus erreichen soll.
                    - Der bisherige Prozess im Projekt klar sein, und wie er mit dem was der Zyklus erreichen soll zusammenhängt.
                    - Der Code vollständig erschlossen sein, sofern er einen Einfluss auf das was der Zyklus erreichen soll hat.
                    - Alle Lücken, die das Erreichen des Ziels des Zyklus erschweren oder verhindern, geschlossen sein.

---

## Phase 2 — Implementieren (nachdem mindestens ein Worker gespawnt ist)


### Step 1 Deliverables und Milestones schneiden.

**Die Arbeit, welche zum Erreichen des Ziels des Zyklus zu erledigen ist, teilt sich in Milestones.**
- Ein erledigter Milestone erzeugt ein Deliverable.
    - Das Deliverable muss getestet und verifiziert sein.
- Milestones werden von Workern parallel oder sequenziell bearbeitet.

```
**M1 of what needs to be done.**
- Erklärung

**M2 of what needs to be done.**
- Erklärung

...

**Mn of what needs to be done.**
- Erklärung
```

### Step 2 Spawn je Milestone

1. Die Session startet auf `main`, führe also `git checkout -b integration` aus.
2. Beim Wechsel auf einen bestehenden Integrationsbranch ist die Branch-Zustandsprüfung zwingend.
    - Führe `git -C <repo> log integration..main --oneline | head -10` aus.
    - Ein nicht leeres Ergebnis heißt, integration hängt hinter main, und Worker würden auf veraltetem Code spawnen.
    - Löse das vor dem Spawnen, durch Rebase von integration auf main oder durch Merge von main nach integration.
3. Worker spawnen.
    3.1 Schreibe den Prompt nach `/tmp/spawn-worker-<project>-<name>.md`.
    3.2 Führe `worker-cli spawn <name> <prompt_file>` aus.

### Step 3 — Cross Model Check je Milestone

1. Lies die Antwort des Workers über `worker-cli response`.
2. Interpretiere ob der Lösungsansatz für den Milestone mit deinem Verständnis konform geht.
    2.1 Falls ja, lasse den Worker implementieren.
    2.2 Falls nein, prüfe kritisch gegen, ob dein Ansatz dem des Workers standhält.
      - Schicke dem Worker deine Korrektur und wiederhole Step 3.

### Step 4 — Implementierung des Workers je Milestone

1. Wenn das WIE, sprich WIE wird der Milestone implementiert, klar ist, gib dem Worker das Go zur Implementierung.
2. Während der Worker arbeitet, gehe gedanklich schon einmal den genauen Code durch, den du vom Worker erwartest.
3. Setze ein worker wait, gehe idle bis der Worker dich weckt.

### Step 5 — Code Review je Milestone

1. Führe `worker-cli response <name>` aus.
2. Lies den vollständigen Diff des Workers über Bash. Der kanonische Command ist:
   ```bash
   git -C <project_root>/.claude/worktrees/<name> diff integration
   ```
   - Beschränke den Diff nicht auf den letzten Commit, denn Code-Review heißt, das gesamte Delta zu lesen.
   - Für den aktuellen Inhalt einer einzelnen Datei nutze `git -C <worktree> show HEAD:<relpath>` oder `cat` über Bash.
3. Bewerte die Arbeit des Workers:
    - Nach funktionaler Korrektheit.
    - Nach Einhaltung der § Code-Standards deines System-Prompts.
5. Werden Probleme gefunden, behandle sie wie einen Milestone und gehe mit diesen Problemen zurück zu Step 2.
6. Besteht der Review, gehe zu Step 6.

### Step 6 — Recap je Milestone

1. Sende `worker-cli send <name> "recap"`.
    - Gib dem Worker IMMER die Area mit in der er seine process-docs schreibt.
    - Der Worker führt seinen Recap autonom durch, sende ihm keine weiteren Anweisungen wie er den Recap durchzuführen hat.
2. Beschränke deinen Review ausschließlich auf die DOCS.md-Dateien.
    - Bewerte die Arbeit des Workers nach Einhaltung der § DOCS.md deines System-Prompts.

### Step 7 — Merge, für alle Milestones auf einmal

1. `worker-cli merge <name>` mergt die Branches des Workers in jedem seiner Repos in den dortigen aktuellen Branch.
    - Der aktuelle Branch ist `integration`, und der Worker samt Worktree bleibt am Leben.
2. Kopiere vor jedem `worker-cli kill` heraus, was nur im Worktree existiert.
    - Gitignorierte Dateien und extrahierte Konfigurationen existieren nur im Worktree.
    - `kill` löscht den Worktree, solche Dateien wären also verloren.

---

## Session-Recap

**Der Session-Recap läuft am Ende der Session.**
- Nur der User kann entscheiden wann eine Session endet.
- Das Ende eines Zyklus ist kein Trigger für einen Session-Recap.

### Phase 1 — RECAP 🔍

1. Gehe gedanklich alle Issues durch, die in der Session bearbeitet wurden.
2. Gehe gedanklich alle Arbeitsaufträge des Users durch, die in dieser Session aufkamen.
3. Gehe gedanklich durch was tatsächlich ausgeführt und abgeschlossen wurde.
4. Präsentiere den Recap in der folgenden Form:

```
**Issues die in der Session bearbeitet wurden**
- Issue X (kann geschlossen werden)
- Issue Y (bleibt offen)
- ...

**Arbeitsaufträge in dieser Session**
- Arbeitsauftrag X
- ...

**Tatsächlich ausgeführt und abgeschlossen in dieser Session**
- ausgeführt und abgeschlossen X
- ...

**Differenz zwischen Arbeitsaufträgen und Abgeschlossenem**
- Vorschlag für neue Issue X
-...
```

🛑 STOP.

### Phase 2 — IMPROVE+CLOSE 🛠️

1. Aktualisiere alle DOCS.md, bei denen du unsicher bist ob sie formal korrekt und auf dem neuesten Stand sind.
2. Schreibe für alles tatsächlich Ausgeführte und Abgeschlossene process-docs, sofern nicht schon von Workern erledigt.
3. Erstelle Issues für die Differenz zwischen Arbeitsaufträgen und Abgeschlossenem.
4. Aktualisiere die Issue-Bodys bearbeiteter Issues bzw. schließe Issues wenn abgeschlossen.
5. Synchronisiere die Docs nach RAG mit `[ -f .rag-docs.json ] && rag-cli update_docs .`.
6. Schließe Git für jedes Repo, das diese Session angefasst hat, einschließlich projektübergreifender Ziele.
    6.1 Führe pro Repo `git checkout main && git merge integration` aus, dann push.
    - Prüfe vor dem Push auf `.claude-plugin/plugin.json`. Existiert sie, nutze `plugin-publish`, ansonsten nutze `git push`.
