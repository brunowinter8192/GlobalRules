# Konvention für das dev/-Verzeichnis

## Ablage der Ausgaben

**Ein dev-Skript schreibt seinen Report nach `dev/<area>/`.**
- Stattdessen auf die Konsole zu schreiben ist nicht erlaubt.
- Innerhalb von `dev/<area>/` geht der Report nach `md/`, `csv/` oder `png/`, gewählt nach Ausgabetyp.
- Die Report-Datei trägt einen beschreibenden Namen, der auf das erzeugende Skript zurückführt.

**Datenausgaben bleiben von Reports getrennt.**
- Skripte erzeugen auch Datenausgaben, etwa Rohkorpora.
   - Datenausgaben gehen in einen eigenen, nach Typ benannten Ordner, zum Beispiel `jsonl/`.
- Datenordner mischen sich nie in `md/`.

**Reports und Daten ordnen sich in `dev/<area>/` nach Thema.**
- Ein dev-Bereich und ein process-docs-Bereich zum selben Thema teilen einen Namen.

## Staging

**Einmal-Skripte leben im Worktree oder in /tmp/ und werden nie gestaged.**
- Baue Forensik und Einmal-Assertions im Worktree oder unter /tmp/.
   - Stage sie beim Merge ausdrücklich nicht.
- Eine Einmal-Assertion, die zum Regressionswächter wird, geht in eine bestehende dev/-Testdatei ein.
   - Eine neue Datei pro Fix ist nicht erlaubt.
