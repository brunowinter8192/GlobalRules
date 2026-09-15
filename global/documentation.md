# Dokumentationshierarchie

## Kernregeln

### Sprache in Dokumentationsdateien

**Jede Dokumentationsdatei wird in Englisch geschrieben.**
- Dokumentationsdateien umfassen DOCS.md und process-docs-Einträge.

### Keine Verweise auf Issues

**Dokumentationsdateien zeigen nie zurück auf Issues.**
- Issues können auf Dokumentationsdateien zeigen, die Richtung bleibt jedoch einseitig.

### RAG-Collections je Projekt

**Es existieren zwei Collections je Projekt.**
- Die docs-Collection hält alle internen Projektdokumente, also DOCS.md und process-docs.
- Neben der docs-Collection gibt es die reference-Collection
    - diese enthält ausschließlich externen Materialien, wie zum beispiel Herstellerdokumentation und Papers.

**Die Collections sind immer gleich benannt:**

Konvention | Beispiel |
|---|---|
`<Project>-docs` | `monitor-cc-docs` |
`<Project>-reference` | `monitor-cc-reference` |

## DOCS.md

**DOCS.md beschreibt die Module eines einzelnen Verzeichnisses.**
   - Die DOCS.md wird mit Änderung des codes ggf. ebenfalls aktualisiert

**Die Form der DOCS.md muss nicht zwingend vollständig ausgefüllt werden**
- lieber einen abschnitt leer lassen als Inhalte einfügen die "weitestgehend" passen

**Ausführlichkeit wird in den process-docs gelebt, die DOCS.md bleiben hingegen schlank.**
- Ein process-docs Eintrag muss ohne code funktionieren
    - die prozesshistory lebt im chat mit dem user und in deinem denken, beides ist nach der session verloren
- Ein docs eintrag wird gebackt vom code
    - alles was im code steht muss nicht in die docs

**DOCS.md wiederholt nie den Code.**
- DOCS.md ist die Vogelperspektive, sie beantworten die frage "wo steht was?" "wo liegen die details?".
    - Was ein Modul im Detail tut, wird in den DOCS.md nicht beantwortet.
- Die einzelnen Konstanten, Parameter, Formeln und Schwellwerte eines Moduls sind aus den docs.md aggresiv auszuschließen

**DOCS.md bewegen sich ausschließlich auf Modulebene.**
- Dokumentation auf Funktionsebene gehört nicht in DOCS.md.
- In DOCS.md entspricht der LOC-Wert jeder Modulüberschrift dem tatsächlichen `wc -l` der Datei.

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

## process-docs

**In process-docs wird festgehalten was ein agent braucht um zu verstehen wie eine entscheidung zustande kam**
- alle Erkenntnisse aus deiner Arbeit die du jetzt sofort einem nachfolger mitgeben würdest 
    - um sicherzustellen das er schneller ist als du
    - um sicherzustellen das er deine fehler nicht wiederholt
    - um sicherzustellen das er eine robuste simple lösung erstellt
    - um sicherzustellen das er sich nicht im kreis dreht und genau erfüllen kann was verlangt ist

**Nutze Beispiele in den process-docs**
- Ein paar Beispiele zeigen das erwartete Verhalten besser als eine ausschweifende abstrakte Erklärung
- Nutze als Beispiele in den process-docs tatsächlich beobachtete Fälle
- Edge cases welche nie beobachtet wurden werden kurz als Hypothese gekennzeichnet und abgehandelt
    - widme Raum dem was greifbar ist, halte nicht greifbares kurz und knapp

**Drücke dich in den process-docs so eindeutig aus wie es nur geht.**
- fange den status quo so exakt ein wie es nur geht

**process-docs sitzen immer im project root**
- Der process-docs-Ordner sitzt immer in der Projektwurzel.
- In der Wurzel trägt er den exakten Namen `process-docs/`.

**Eine process-docs Datei pro Session**
- Jeder Agent schreibt über seine gesamte Lebensdauer genau eine process-docs-Datei.
   - Eine process docs datei wird stetig erweitert, solange die session besteht
- Ausnahme, wenn merhere `process-docs/<area>/` berührt werden
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
