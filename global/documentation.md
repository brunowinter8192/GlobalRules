# Dokumentationshierarchie

## Kernregeln

### Sprache (NICHT VERHANDELBAR)

**Jede Dokumentationsdatei wird in Englisch geschrieben.**
   - Dokumentationsdateien umfassen DOCS.md, process-docs-Einträge, dev/-Reports und Code-Kommentare.

### Abschnitte sind optional

**Weglassen, nicht auffüllen.**
- Jeder Abschnitt in DOCS.md und in einem process-docs-Eintrag ist optional.
   - Ein optionaler Abschnitt ohne Inhalt wird weggelassen.
   - Weglassen schlägt es, ein Feld zu füllen, nur weil die Vorlage es hat.

### Keine Verweise auf Issues

**Docs zeigen nie zurück auf Issues.**
- Dateien unter process-docs verweisen nie auf Issues.
- Issues zeigen auf Docs, und die Richtung bleibt einseitig.

### RAG-Sammlungsschichten

**Zwei Sammlungen pro Projekt.**
- Die docs-Sammlung hält alle internen Projektdokumente, also DOCS.md und process-docs.
- Neben der docs-Sammlung hält die reference-Sammlung alle externen Quellen, etwa Herstellerdokumentation und Papers.

**Kanonische Benennung:**

| Schicht | Konvention | Beispiel |
|---|---|---|
| docs | `<Project>-docs` | `monitor-cc-docs` |
| reference | `<Project>-reference` | `monitor-cc-reference` |

## docs

**DOCS.md ist die Modulkarte.**
- Eine DOCS.md beschreibt die Module ihres Verzeichnisses.
   - Diese Beschreibung ist die einzige Dokumentation, die laufend aktualisiert wird.

**Ausführlichkeit gehört zu process-docs, und DOCS.md bleibt schlank.**
- Ein process-docs-Eintrag trägt jede Länge, die sein Autor braucht.
- Inhalt, der nicht in die schlanke Form von DOCS.md passt, geht stattdessen nach process-docs.

**DOCS.md wiederholt nie den Code.**
- DOCS.md ist die Vogelperspektive und beantwortet, welche Module für eine gegebene Frage relevant sind.
- Was ein Modul im Detail tut, wird in DOCS.md nicht beantwortet.
- Die einzelnen Konstanten, Parameter, Formeln und Schwellwerte eines Moduls bleiben draußen.
   - Benenne stattdessen die Gruppe, die sie bilden.

### Ablage

**Eine DOCS.md pro Modulverzeichnis.**
- Die Datei liegt in dem Verzeichnis, das die `.py`-Module hält, die sie dokumentiert.

### DOCS.md-Format

```markdown
# <dir>/

## Role
One paragraph, 50 words maximum — WHAT this directory does in the bigger picture (not HOW), when to touch it, when NOT to touch it.

## Public Interface
What `__init__.py` exports. One line per export. If `__init__.py` is empty: say so and state the actual entry path (e.g. "loaded via `mitmproxy -s`").

## Flow
3-5 lines: data in → processing → data out.

## Modules

### <module>.py (<LOC> LOC)

**Purpose:** one sentence, 25 words maximum.
**Reads:** data sources (shared state, files, stdin).
**Writes:** outputs (stdout, files, shared state, mutated state).
**Called by:** list of files/packages. Empty list = DEAD CODE, flag explicitly.
**Calls out:** external package dependencies (not stdlib, not `constants`/`utils`).

---

## State
Which module owns the state, who mutates, who reads.
```

**Nur auf Modulebene.**
- Dokumentation auf Funktionsebene gehört nicht in DOCS.md.
- In DOCS.md entspricht der LOC-Wert jeder Modulüberschrift dem tatsächlichen `wc -l` der Datei.

## process docs

**Ein paar gute Beispiele schlagen eine erschöpfende Liste.**
- Ein paar unterschiedliche Beispiele zeigen das erwartete Verhalten besser als eine Halde von Randfällen.
- Ein Beispiel verdient seinen Platz nur, wenn es zeigt, WIE man entscheidet.
   - Nur zu zeigen, dass ein Fall existiert, verdient keinen Platz, deshalb fällt so ein Beispiel raus.

**Eindeutige Benennung.**
- Mache impliziten Kontext explizit.
- Expliziter Kontext heißt Namen, die der Leser nicht falsch auflösen kann.
   - Ein Name wie `user_id` löst sauber auf, wo `user` es nicht tut.

**Artefakte werden für einen anderen Agenten geschrieben**
- Ein Artefakt ist alles, was du außerhalb des Chat outputs der sich direkt an den user richtet, liest oder erzeugst.
   - Artefakte umfassen process-docs.
- gehe immer davon aus das der lesende Agent mit 0 Context beginnt wenn er dein artefakt liest.

**Immer in der Projektwurzel, immer process-docs genannt.**
- Der process-docs-Ordner sitzt immer in der Projektwurzel.
- In der Wurzel trägt er den exakten Namen `process-docs/`.
- Was der Ordner festhält, ist, wie Dinge untersucht und entschieden wurden.

**Alles, was nicht in die schlanke Form von DOCS.md passt, wird hier festgehalten.**
- Prozesshistorie gehört hierher, also Daten und was extrahiert, aufgeteilt, ersetzt oder umbenannt wurde.
- Belege gehören hierher, also Verifikationsberichte, gemessene Ergebnisse und Begründungen.
- Modulspezifische Tretminen und Absicherungen kalibrierter Werte gehören hierher.
- Detail, das direkt auf den Code verweist, gehört hierher.
- Deine eigene Argumentation gehört hierher, wann immer ein folgender Agent sie nutzen kann.

**Ein Eintrag ist ein Abschnitt in der eigenen Datei des Autors, nie eine Datei für sich.**
- Eine Datei sammelt so viele Einträge an, wie ihr Autor Themen hat.

**Eine Datei pro Autorensession, pro Bereich.**
- Ein Worker schreibt über seine gesamte Lebensdauer genau eine process-docs-Datei.
   - Jeder Recap dieses Workers hängt an dieselbe Datei an.
- Eine Main-Session schreibt genau eine process-docs-Datei.
- Inhalt, der mehrere Bereiche umspannt, bekommt eine Datei pro Bereich, jede in ihrem eigenen `process-docs/<area>/`.
- Keine andere process-docs-Datei wird jemals angefasst, egal was sie enthält.
   - Jede Main-Session und jeder Worker hat die eigene Datei als einzigen beschreibbaren Bereich.
   - Ein gefundener Fehler, eine veraltete Behauptung oder ein Widerspruch in einer anderen Datei wird in der eigenen Datei festgestellt, nie in der anderen korrigiert.

**Einmal schreiben, nicht pflegen.**
- Eine process-docs-Datei ist eine datierte Momentaufnahme, geschlossen wenn die Session ihres Autors endet, und danach nie mehr angefasst.
   - Neue Arbeit bekommt eine NEUE Datei, statt die alte anzufassen.

**Keine Gegenwartsbehauptungen über den "aktuellen" Stand.**
- Ein Eintrag behauptet nie einen Produktionsstand in der Gegenwart, wie "X ist der Produktionswert".
- Produktionsstand wird stattdessen auf sein Datum bezogen gerahmt, wie "Stand 2026-06 zeigte der Sweep X".

**Strukturiert, keine chaotische Halde.**
- Das Thema entscheidet den Ordner, also ordnen sich Einträge in `process-docs/<area>/`-Unterordner.
   - Innerhalb eines Unterordners ist die Dateibenennung frei, also funktionieren datumsbasierte und zweckbasierte Namen beide.

**Ein Bereich ist eine Arbeitslinie.**
- Ein Bereich läuft über Sessions hinweg und sammelt Einträge an.
- Der Name des Bereichs wird identisch von Issues, `process-docs/<area>/` und `dev/<area>/` verwendet.

**Methoden und Antworten innerhalb eines Bereichs dürfen sich vollständig ändern.**
- Ein vollständiger Wechsel des Ansatzes setzt den Bereich fort, denn ein Schwenk ist keine neue Frage.

**Querverweise zeigen auf BEREICHE, nie auf einzelne Einträge.**
- Ein process-docs-Eintrag darf keine andere process-docs-Datei über ihren Pfad referenzieren.
- Der Ordner eines anderen Bereichs, `process-docs/<area>/`, darf referenziert werden, und das ist gewollt.

**Belege bleiben im Text.**
- Nenne das Kernergebnis einer Messung in der Prosa selbst.
   - Das Kernergebnis heißt die Zahl, die Datensatzgröße und der Befund.
- Ein Link auf einen dev/-Report darf die Behauptung stützen, bleibt aber optionale Lektüre.
