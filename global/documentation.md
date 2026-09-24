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
- Neben der docs-Collection gibt es die reference-Collection.
    - Diese enthält ausschließlich externe Materialien, wie zum Beispiel Herstellerdokumentation und Papers.

**Die Collections sind immer gleich benannt:**

Konvention | Beispiel |
|---|---|
`<Project>-docs` | `monitor-cc-docs` |
`<Project>-reference` | `monitor-cc-reference` |

## DOCS.md

**DOCS.md beschreibt die Module eines einzelnen Verzeichnisses.**
   - Die DOCS.md wird mit Änderung des Codes ggf. ebenfalls aktualisiert.

**Die Form der DOCS.md muss nicht zwingend vollständig ausgefüllt werden.**
- Lieber einen Abschnitt leer lassen als Inhalte einfügen die "weitestgehend" passen.

**Die Details kommen in die process-docs, die DOCS.md bleiben schlank.**
- Ein process-docs Eintrag muss ohne Code funktionieren.
    - Die Prozesshistorie steckt im Chat mit dem User und in deinem Denken, beides ist nach der Session weg.
- Ein Docs-Eintrag ist backed by Code.
    - Alles was im Code steht muss nicht in die Docs.

**DOCS.md wiederholt nie den Code.**
- DOCS.md ist die Vogelperspektive und beantwortet die Frage "wo steht was?" und "wo liegen die Details?".
    - Was ein Modul im Detail tut, wird in den DOCS.md nicht beantwortet.
- Die einzelnen Konstanten, Parameter, Formeln und Thresholds eines Moduls sind aus den DOCS.md aggressiv auszuschließen.

**DOCS.md beschreiben ausschließlich Module.**
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

**In process-docs wird festgehalten was ein Agent braucht um zu verstehen wie eine Entscheidung zustande kam.**
- Alle Erkenntnisse aus deiner Arbeit die du jetzt sofort einem Nachfolger mitgeben würdest.
    - Um sicherzustellen dass er schneller ist als du.
    - Um sicherzustellen dass er deine Fehler nicht wiederholt.
    - Um sicherzustellen dass er eine robuste simple Lösung erstellt.
    - Um sicherzustellen dass er sich nicht im Kreis dreht und genau erfüllen kann was verlangt ist.

**Nutze Beispiele in den process-docs.**
- Ein paar Beispiele zeigen das erwartete Verhalten besser als eine ausschweifende abstrakte Erklärung.
- Nutze als Beispiele in den process-docs tatsächlich beobachtete Fälle.
- Edge Cases welche nie beobachtet wurden werden kurz als Hypothese gekennzeichnet und abgehandelt.
    - Widme Raum dem was greifbar ist, halte nicht Greifbares kurz und knapp.

**Drücke dich in den process-docs so eindeutig aus wie es nur geht.**
- Fange den Status quo so exakt ein wie es nur geht.

**process-docs sitzen immer im project root.**
- Der process-docs-Ordner sitzt immer in der Projektwurzel.
- In der Projektwurzel trägt der Ordner den exakten Namen `process-docs/`.

**Es wird eine process-docs Datei pro Session erstellt.**
- Jeder Agent schreibt über seine gesamte Lebensdauer genau eine process-docs Datei.
   - Eine process-docs Datei wird stetig erweitert, solange die Session besteht.
- Ausnahme, wenn mehrere `process-docs/<area>/` berührt werden, dann darf für jede Area eine Prozessdatei erstellt werden.
- Keine andere process-docs Datei wird jemals editiert, egal was sie enthält.
   - Jeder Agent hat die eigene Datei als einzigen beschreibbaren Bereich.
   - Ein gefundener Fehler, eine veraltete Behauptung oder ein Widerspruch in einer anderen process-docs Datei wird in der eigenen process-docs Datei festgestellt, nie in der anderen korrigiert.

**Keine Gegenwartsbehauptungen über den "aktuellen" Stand in einer process-docs Datei.**
- Nutze nicht das Wording aktueller Stand in den process-docs, nutze stattdessen ein Datum.

**Querverweise zeigen auf Areas, nie auf einzelne process-docs Dateien.**
- Eine process-docs-Datei darf keine andere process-docs-Datei über ihren Pfad referenzieren.
    - Referenziere stattdessen die Area in der die gewünschte process-docs Datei liegt.
