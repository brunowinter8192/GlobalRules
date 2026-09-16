# Workers

## Kernregeln

### Du editierst keinen Quellcode

**ALLE Quellcode-Änderungen laufen über Worker.**
- Jede `.py`-, `.sh`-, `.js`-, `.ts`- oder andere Quelldatei geht an einen Worker.

**Docs und Skills darfst du direkt editieren.**
- Du darfst Skills und alle Dokumentationsdateien (DOCS.md und process-docs) direkt editieren.

### Schreiben von Dokumentationsdateien

**Wer den Input hat, schreibt die Dokumentationsdatei.**
- Den Chat mit dem User kannst nur du sehen, alles was daraus erwächst dokumentierst du direkt
- Den Chat mit dem Worker können du und der Worker sehen
    - Delegiere das Schreiben von Dokumentation, die sich auf euren Chat bezieht, an den Worker

### Externe Quellen

**Mit externen Quellen werden RAG, GitHub, Reddit und Web bezeichnet.**
- Der Worker hat nativ keinen Zugang zu externen Quellen, und das ist auch nicht vorgesehen.

**Der Worker liest nur was du ihm zur Verfügung stellst.**
- Klone Repos oder einzelne Files und stelle sie dem Worker zur Verfügung
- Baue Kernaussagen von Reddit-Threads in den Prompt ein
- Baue Kernaussagen aus dem Web in den Prompt ein
- Es gilt: Alles Externe, von dem du denkst dass der Worker es für die korrekte Ausführung seiner Aufgabe braucht, musst du ihm zur Verfügung stellen
    - Wie du das praktisch machst, bleibt dir überlassen
    - Der Worker darf jedoch keine CLI-Tools jedweder Art zur externen Informationsbeschaffung nutzen

### Bereich der Worker

**Jeder Worker muss in einem Worktree arbeiten.**

**Projektübergreifende Arbeit nutzt zwei Worktrees.**
- Der Worker ist eine normale Claude-Code-Session
    - Er ist in der Lage, Arbeit in einem beliebigen Projekt zu verrichten
- Für Arbeit in einem anderen Projekt erzeugst du den Ziel-Worktree nach dem Spawn.
   - Der Worker macht seine Arbeit dann dort.
- Der Worker spawnt also im aktuellen Projekt und arbeitet im Worktree des Zielprojekts.

### Lebenszyklus eines Workers

**Worker werden für Folgetasks wiederverwendet**
- Verwende einen Worker für alles in seinem thematischen Bereich wieder
    - Wenn ein Worker Teile des Wissens besitzt, die mit der Folgetask zusammenhängen, verwende denselben Worker erneut
- Ein paralleler Worker braucht einen Grund:
   - eine ausdrückliche Bitte des Users
   - eine völlig orthogonale neue Aufgabe
   - eine völlig parallel ableistbare Aufgabe
   - ein toter aktueller Worker

### Ein Worker stirbt

**Ein toter Worker hat für den laufenden Meilenstein nichts committet.**
- Die Commits, die es noch gibt, stammen also aus früheren Meilensteinen.

**Stirbt ein Worker während einer Aufgabe, übernimmt ein Nachfolge-Worker.**
1. Führe zuerst `worker-cli capture <name>` aus und lies das Pane.
2. Merge abgeschlossene, aber nicht gemergte Commits vom toten Branch nach `integration`.
3. Kille den toten Worker.
4. Spawne den Nachfolger mit einem Prompt aus Dateien, dem Meilenstein und der Stelle, wo er aufnimmt.
5. Prüfe die erste Antwort des Nachfolgers gegen den Punkt, wo der tote Worker aufhörte, wie in Phase 2 Step 2.

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
- worker token kosten nicht viel, dennoch sind die worker in der lage komplexe zusammenhänge zu erfassen
    - zusammenhänge werden aber in der regel nur erfasst wenn der worker files komplett liest
    - Du hast die Macht über ein unendliches Worker-Kontingent, nutze es um neue Blickwinkel zu erhalten, du bist angehalten Worker für ausschweifende Suchen zu nutzen

---

## Sessionzyklus

**Zeige dem User im Chat bei jedem Step- und Phasenwechsel den neuen Step und die neue Phase an**

- `📋 Phase 1 — Step 1: Session Scope`
- `📋 Phase 1 — Step 2: Process Investigation`
- `📋 Phase 1 — Step 3: Code Investigation & Gap Analysis`
- `📋 Phase 1 — Step 4: Deliverables & Milestones`
- `🔨 Phase 2 — Step 1: Dispatch`
- `🔨 Phase 2 — Step 2: Evaluate`
- `🔨 Phase 2 — Step 3: Go`
- `🔨 Phase 2 — Step 4: Review`
- `🔨 Phase 2 — Step 5: Recap`
- `🔨 Phase 2 — Step 6: Merge`

---

## Phase 1

**Die Steps laufen sequenziell mit einem Gate nach jedem Step.**
- Stelle nach jedem Step die Befunde dar und warte auf Anmerkungen, bevor du fortfährst.

**In der Planungsphase ist die Chatausgabe nicht auf Exchanges und Action Frames begrenzt.**
- Gib aus, was die Vorlage des Steps sagt.

**Arbeite extrem eng mit dem User, keine Arbeit geht über seinen Prompt hinaus.**
- Tu genau das, was der User verlangt, begrenzt auf das, wovon du sicher bist, dass sein Prompt es verlangt hat.
- Ein Exchange trägt nur Schlussfolgerungen, die eindeutig an den Prompt des Users anknüpfen.
- Erkläre nur, was genau mit dem Prompt des Users zu tun hat.

### Step 1 — Session Scope

- Wiederhole in eigenen Worten, was der User will.

🛑 STOP — Ask for remarks.

### Step 2 — Prozessuntersuchung

**Durchsuche die Prozesshistorie über RAG in ZWEI Durchgängen und verfeinere die Abfrage dazwischen.**
- Jeder Durchgang führt `search` auf `<Project>-docs` aus, begrenzt auf die Prozessschicht und nie auf die Codekarte.
- Der zweite Durchgang wiederholt nie die Abfrage des ersten, denn die Treffer des ersten sagen dir, wonach du fragen musst.
   - Wähle die Chunks, die am besten trafen, und nimm ihr Vokabular in die verfeinerte Abfrage.
- Der Durchgang, mit dem du nicht beginnst, ist nicht optional, und ihn zu überspringen ist keine Ermessensfrage.
   - Ein auf den Bereich begrenzter Durchgang kann strukturell keine benachbarte Arbeit hervorbringen, wie gut er auch formuliert ist.
   - Ein Mechanismus wird routinemäßig in einem Bereich gelöst und von dem Bereich, in dem du bist, nur geerbt.

**Mit einem Issue kommt der Bereich aus dem `Area:`-Feld des Issues, und der Bereichsdurchgang läuft zuerst.**
1. Abfrage begrenzt mit `--document 'process-docs/<area>/%'`.
2. Entscheide, welche Chunks am besten trafen.
3. Verfeinere die Abfrage aus diesen Chunks.
4. Führe die verfeinerte Abfrage bereichsübergreifend aus, begrenzt mit `--document 'process-docs/%' --exclude 'process-docs/<area>/%'`.

**Ohne ein Issue ist der Bereich noch offen, deshalb läuft der bereichsübergreifende Durchgang zuerst.**
1. Abfrage begrenzt mit `--document 'process-docs/%'`.
2. Entscheide, welche Chunks am besten trafen, und lass sie den EINEN Bereich benennen, zu dem diese Arbeit gehört.
3. Verfeinere die Abfrage aus diesen Chunks.
4. Führe die verfeinerte Abfrage begrenzt mit `--document 'process-docs/<area>/%'` aus.

**Jeder Treffer, der dein Prozessverständnis trägt, wird zuerst mit `read_document` erweitert.**
- Ein Treffer trägt das Verständnis, sobald ein Satz deiner Darstellung auf ihm ruht.
- Nur die Treffer zu erweitern, die du subjektiv als wichtig einordnest, ist nicht der Standard.
- Ein nackter Such-Snippet ist nie eine ausreichende Grundlage für eine Aussage an den User.
- N Treffer in die Darstellung zu tragen heißt daher N Erweiterungen, bevor du sie schreibst.

**Die Bereichsbeurteilung ist ein zwingender Teil der Ausgabe dieses Steps.**
- Das ist ein User-Gate, damit der User hier eingreifen kann.
   - Hinter dem Gate ist der Bereich für die Session festgelegt.
   - Wenn mitten in der Session ein anderer Bereich richtig erscheint, markiere es statt still zu wechseln.

NEUER Bereich — IRGENDEINES genügt:

- Baut ANDERE Arbeit ebenfalls auf diesem Bereich auf?
   - Ein Ja macht den Bereich zu einer gemeinsamen Basis statt zu einem privaten Vorgänger.
- Zieht die Arbeit neben diesem noch aus ANDEREN Bereichen?
- Hängt die Arbeit an KEINEM bestehenden Bereich?

BESTEHENDER Bereich (ihn fortsetzen) — ALLE drei müssen gelten:

- Hängt die Arbeit an den Einträgen dieses Bereichs?
- Ist das Fundament dieses Bereichs das Fundament DIESER Fortsetzung und keiner anderen?
- Zieht die Arbeit allein aus diesem EINEN Bereich?

**Vorlage**

```
**Last step of the process prior to the current scope.**
- elaboration

🛑 STOP — Ask for remarks.
```

### Step 3 — Codeuntersuchung und Lückenanalyse

**Stufe 1, lies den Code.**
- Die Modulkarte ist der Einstieg, und der Quellcode ist das Einzige, was direkt gelesen wird.

1. Abfrage `search` auf `<Project>-docs`, begrenzt mit `--exclude 'process-docs/%'`.
2. Entscheide, welche Module die Treffer als relevant benennen, und lies jede Datei, die der Worker anfassen wird.
3. Verfeinere die Abfrage aus diesen Dateien und führe sie gegen denselben Bereich aus.
4. Lies die weiteren Dateien, die der zweite Durchgang hervorbringt, bis der Plan des Workers beurteilbar ist.

**Stufe 2, Lückenanalyse.**
- Das Ziel und die angefassten Dateien sind nach Stufe 1 schon klar.
- Eine Lücke ist eine Stelle, an der noch etwas schiefgehen kann.
- Eine Lücke schließt sich auf genau zwei Wegen.
   - Der erste Weg ist eine Messung, also eine dev/-Sonde, die das echte Verhalten hervorbringt.
   - Der zweite Weg ist eine externe Ressource, also Wissen, das nicht im Projekt liegt.
- Gehe die möglichen Stolpersteine durch und benenne für jeden, welcher der zwei ihn schließt.
   - Wo beide funktionieren würden, bevorzuge die externe Ressource.
- Die externe Ressource braucht deine Handlung, markiere sie also dem User, der sie beschafft.

**Externe Ressourcen, benenne und markiere sie ohne zu ringen.**
- Wäge nicht ab, ob es sich lohnt, externe Quellen hereinzuholen.
   - Stell dir vor, jede Ressource der Welt ist verfügbar, und eine Markierung schließt die Lücke.
- Benenne aus dem Trainingswissen die Art von Quelle, die dein mentales Modell festigen würde.
   - Bei Communities wie Reddit beurteile, ob das Thema dort besprochen werden könnte.
- Du wirst das exakte Repo oder den exakten Post nicht kennen, und das ist in Ordnung.
   - Die Beurteilung ist, ob sich diese Art von Suche auszahlen würde.

**Jede Lücke wird als ein Kanal plus die Punkte dargestellt, die du davon willst.**
- Der Kanal ist genau einer von `gh`, `web` oder `reddit`, und nie eine Domain oder eine URL.
   - `gh` deckt Quellcode, Patch-Sets und Issue-Threads ab.
   - `web` deckt offizielle Dokumentation und Referenzlisten ab.
   - `reddit` deckt Praxiserfahrung ab, die keine Dokumentation trägt.
- Die Punkte unter einer Lücke sagen, WAS du aus diesem Kanal willst, nicht wo er sitzt.
- Eine Lücke, die nur du oder der User beantworten kann, benennt diese Person statt eines Kanals.

**Vorlage**

```
Gap 1 — <gap in one line> — gh
- elaboration

Gap 2 — <gap in one line> — web
- elaboration

Gap 3 — <gap in one line> — reddit
- elaboration

🛑 STOP — Ask for remarks.
```

### Step 4 — Liefergegenstände und Meilensteine

**Schneide so viele Meilensteine wie nötig.**
- Ein Meilenstein ist eine logisch abgegrenzte Einheit, unabhängig committebar und verifizierbar, endend in einem Liefergegenstand.

**Vorlage**

```
**Big picture of what needs to be done.**
- elaboration

**M1 of what needs to be done.**
- elaboration

**M2 of what needs to be done.**
- elaboration

...

**Mn of what needs to be done.**
- elaboration

🛑 STOP — Ask for remarks.
```

---

## Phase 2 — Implementieren (nachdem mindestens ein Worker gespawnt ist)

**Arbeite extrem eng am Worker.**
- Die Phase mit dem User ist vorbei.
- Ziehe den User nur für kritische, Entscheidung verlangende Exchanges hinzu.

### Step 1 — Dispatch

**Beauftrage EINEN Meilenstein zur Zeit, nie den ganzen Plan.**
- Reiche dem Worker den Meilenstein als abstrakte Aufgabe plus die benannten Dateien.

**Stufe 1, der Integrationsbranch.**
- Worker mergen auf `integration` und nie auf `main`.

1. Die Session startet auf `main`, führe also `git checkout -b integration` aus oder wechsle auf den bestehenden.
2. Beim Wechsel auf einen bestehenden Integrationsbranch ist die Branch-Zustandsprüfung zwingend. Führe `git -C <repo> log integration..main --oneline | head -10` aus. Ein nicht leeres Ergebnis heißt, integration hängt hinter main, und Worker würden auf veraltetem Code spawnen. Löse das vor dem Spawnen, durch Rebase von integration auf main oder durch Merge von main nach integration. Auf veraltetem integration zu bleiben braucht ein ausdrückliches OK des Users.
3. Worker spawnen, und ihre Worktrees zweigen von `integration` ab.
4. `worker-cli merge` mergt nach `integration`.
5. Am Sessionende synchronisiert `git checkout main && git merge integration` integration nach main.

**Stufe 2, Prompt-Struktur und Spawn.**
- Der Prompt beschreibt WAS, und der Worker findet das WIE selbst heraus.
- Jeder Prompt entspricht genau dem, was mit dem User vereinbart wurde.
   - Extras am Weg und Variablen, nach denen der User nicht gefragt hat, sind nicht erlaubt.

| MUSS enthalten | DARF NICHT enthalten |
|---|---|
| Die Aufgabe abstrakt beschrieben, also das Problem und das gewünschte Ergebnis. | Exakten zu schreibenden Code. Der Worker findet seine eigene Implementierung. Externer Referenzcode von außerhalb des Projekts ist die eine Ausnahme, und du lieferst ihn. |
| Die Dateien und Verzeichnisse, die du definitiv als relevant befunden hast. Sie sind ein Startsatz und kein Zaun. Ergänze alle process-docs-Einträge, die der Worker zum Kontext lesen soll. | Ursachenhypothesen, die als Fakten dargestellt sind. |
| Den Worktree-Pfad als Arbeitsplatz, formuliert wie "Your worktree is `<project>/.claude/worktrees/<name>/`. Work, test, and commit here." | Implementierungsdetails, die den Ansatz des Workers einschränken. |
| Den ausdrücklichen Negativbereich, formuliert wie "Do NOT add features or improvements beyond the listed deliverables." | Eine Tool-Beschränkung, die weiter formuliert ist, als der Hook sie erzwingt. |
| Die aufgabenspezifischen Punkte der Completion Checklist, also die Verifikationspunkte, die der Worker am Ende ausgibt. | |
| Den Satz "You are a WORKER." | |

Dann spawnen:
1. Schreibe den Prompt nach `/tmp/spawn-worker-<project>-<name>.md`.
2. Führe `worker-cli spawn <name> <prompt_file> <project_path> [model]` aus. Der Worktree ist der Standard, lass `--no-worktree` also weg.
3. Bewaffne sofort den Wake-up, in der Form, die die Wake-up-Schleife beschreibt.

### Step 2 — Bewerten

**Vergleiche den Plan des Workers mit deinem eigenen mentalen Modell aus Phase 1.**
- Nach dem Dispatch liest der Worker Dateien im Worktree und berichtet Befunde plus Ansatz.
   - Lies den Bericht über `worker-cli response`.
- Prüfe auf dieselbe Ursache, dieselben Zieldateien und denselben Ansatz.
- Bei Übereinstimmung sende "Go, implement it."
- Bei Abweichung jeder Art musst du selbst prüfen.
   - Beurteile, ob die Abweichung des Workers von deinem mentalen Modell tatsächlich richtig ist.
   - Wenn sie richtig ist, gib Go.
   - Wenn sie falsch ist, sende genau wo und warum, und bleib bei Step 2.
- Vorschläge des Workers ungeprüft zu übernehmen ist verboten.
   - Einen Plan mit "sieht gut aus" durchzuwinken ist ebenfalls verboten.

### Step 3 — Go und Implementierung

- Der Worker implementiert, nachdem er Go erhalten hat.

### Step 4 — Review

**Nachdem der Worker idle geht, reviewe VOR dem Mergen.**

#### Code-Review (ZWINGEND)

1. Führe `worker-cli response <name>` aus.
2. Lies den vollständigen Diff des Workers über Bash. Das kanonische Command ist:
   ```bash
   git -C <project_root>/.claude/worktrees/<name> diff integration
   ```
   Beschränke den Diff nicht auf den letzten Commit, denn Code-Review heißt, das gesamte Delta zu lesen. Für den aktuellen Inhalt einer einzelnen Datei nutze `git -C <worktree> show HEAD:<relpath>` oder `cat` über Bash.
3. Prüfe Korrektheit, Einhaltung bestehender Muster und das Fehlen von Regressionen.
4. Prüfe jeden angefassten `DOCS.md`-Hunk gegen § DOCS.md-Format, und beurteile ihn gegen dieses Format, nie gegen die benachbarten Einträge.
5. Werden Probleme gefunden, behandle sie als Review-Meinungsverschiedenheit.
6. Besteht der Review, gehe zu Step 5.

**Der Review ist nicht überspringbar, auch nicht bei Ad-hoc- oder Einzeiler-Merges.**
- Frage dich vor jedem `worker-cli merge`, ob du den Diff in dieser Session ausgeführt und gelesen hast.
   - Wenn nicht, stopp und führe zuerst den Diff aus.


#### Review-Meinungsverschiedenheiten

**Eine Review-Meinungsverschiedenheit wird genau wie eine Abweichung in Step 2 behandelt.**
- Dieselbe Prüfung gilt, und du schreibst keinen Patch vor.

### Step 5 — Recap (ZWINGEND nach jedem Meilenstein)

**Nachdem Step 4 sauber abgeschlossen ist, sendest DU den Recap-Auslöser.**
- Sende `worker-cli send <name> "recap"` nach jedem Meilenstein, ohne Ausnahme.
- Der Auslöser ist deiner, und der Worker fährt seinen eigenen Recap-Durchlauf, begrenzt auf seinen Meilenstein.
- Der Recap bündelt die DOCS.md-Aktualisierung und den process-docs-Eintrag in einen Commit.
   - Es passiert jetzt, denn der Worker hat den Aufgabenkontext noch im Kopf.
- Stirbt der Worker mitten im Recap, beendest du den Recap selbst.
- Dokumentationsdrift auf den Recap am Sessionende zu verschieben ist nicht erlaubt.

**Die Ausgabe ist ein Recap-Commit, in den Merge eingefaltet.**
- Der Worker committet einen Recap-Commit namens `docs: recap for <task>`.
   - Er berichtet die angefassten Dateien und die Doc-Aktualisierungen.

### Step 6 — Merge

**Kopiere heraus, was nur im Worktree lebt, bevor du mergst.**
- Gitignorierte Dateien und extrahierte Konfigurationen existieren nur im Worktree.
   - Der Merge löscht den Worktree, solche Dateien wären also verloren.

**`worker-cli merge <name> [project_path]` mergt den Branch in den aktuellen Branch.**
- Der aktuelle Branch ist `integration`, und der Worker bleibt am Leben.
- Bei einem projektübergreifenden Worker ist `project_path` zwingend.


---

## Session-Recap

**Der Session-Recap läuft ganz am Ende, nur auf den ausdrücklichen Auslöser des Users.**
- Er ist vom Worker-Zyklus entkoppelt, und der User entscheidet, wann er passiert.
- Frage nie danach und schlage ihn nie vor.

**Dein Session-Recap umfasst NUR Dateien, die du direkt angefasst hast.**

### Phase 1 — RECAP 🔍

**Die Issue-Bewertung umfasst nur Issues, die diese Session angefasst hat.**
- Lass die übrigen unangetastet.
- Entscheide für jedes angefasste Issue zwischen Schließen und Offenhalten.
- Erstelle ein neues Issue nur für eine eigenständige Aufgabe, die diese Session aufkam und offen bleibt.

**Leerer Teller, erfasse jeden nicht ausgeführten offenen Punkt vor dem Schließen.**
- Jeder offene Punkt aus dem ursprünglichen Plan, der nicht ausgeführt wurde, wird erfasst.
   - Meist ist diese Erfassung ein process-docs-Eintrag.
   - Ein Issue ist nur richtig, wenn der Punkt eine eigenständige Aufgabe für sich ist.

**Vorlage**

```
**Issues touched this session.**
- elaboration

**Open items captured.**
- elaboration

**Doc files written or edited in the improve phase.**
- elaboration

🛑 STOP — Ask for remarks.
```

### Phase 2 — IMPROVE+CLOSE 🛠️

**Ein Durchlauf, ohne Stopps.**
1. Führe die Chat-Zusammenfassung aus, schreibe also die benannten Doc-Dateien und mache die Issue-Hygiene genau wie dargestellt.
2. Synchronisiere die Docs nach RAG mit `[ -f .rag-docs.json ] && rag-cli update_docs .`.
3. Schließe Git für jedes Repo, das diese Session angefasst hat, einschließlich projektübergreifender Ziele. Führe pro Repo `git checkout main && git merge integration` aus, dann `gcommit "<message>"`, dann push. Prüfe vor dem Push auf `.claude-plugin/plugin.json`. Existiert sie, nutze `plugin-publish`, ansonsten nutze `git push`.
