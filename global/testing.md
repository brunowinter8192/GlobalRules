# Test und Verifikation

## Was ein Test ist

**Ein Test hängt nur von Faktoren ab, die du kontrollierst, ohne Abhängigkeit von der Umgebung.**
- Ihn 10 oder 100 Mal hintereinander zu wiederholen ergibt jedes Mal dasselbe Ergebnis.
- Ihn in einem Monat oder in einem Jahr laufen zu lassen ergibt ebenfalls dasselbe Ergebnis.

## Wann ein Test erforderlich ist

**Ein Test ist genau dann erforderlich, wenn eine Implementierung Verhalten ändert.**
- Die Anwendung zeigt Verhalten A, und die Implementierung bewegt es zu A.1 oder B.
- Der Test zeigt, dass Verhalten A.1 oder B der Spezifikation entsprechen soll.

**Der Test einer Änderung beweist auch, dass die Aufrufer des geänderten Codes sich wie vorher verhalten.**
- Jeder Aufrufer wird über Import-Grep und die DOCS.md-Karte gefunden.
- Die Aufruferprüfung ist in den Test jeder Änderung einzeln eingewoben, deshalb braucht es keine gepflegte Suite.
- Ein kaputter Aufrufer ist eine verursachte Verhaltensänderung und gehört damit zur Aufgabe.

**Ein Test deckt die Funktionalität ab, die der Meilenstein liefert, und nichts darüber hinaus.**
- Ein Randfall, den niemand in echten Daten beobachtet hat, bekommt keinen Test.
- Ein Testfall, den der Implementierer sich ausdenkt, ist keine Beobachtung.

## Was eine Verifikation ist

**Eine Verifikation entspricht genau der Produktionsumgebung.**
- Sie läuft einmal, wenn Tests keinen Erkenntnisgewinn mehr bringen.

**Eine Verifikation, die das beabsichtigte Verhalten A.1 oder B nicht zeigt, geht zurück in die Implementierung.**
- Zurück zur Implementierungsänderung, erneut testen, dann erneut verifizieren.
- Eine Verifikation, die zweimal scheitert, stoppt alle Handlungen sofort, und du meldest.

## Beweislast

**Komplexität, die es in die Produktion schafft, führt auf einen in echten Daten beobachteten Fehler zurück.**
- Ein Fixture, geschrieben von der Person, die die Absicherung fordert, zählt nie als diese Beobachtung.

**Eine vorgeschlagene Absicherung bringt einen gemessenen Preis und ein gemessenes Risiko mit.**
- Ein gemessener Verlust wird nie gegen einen hypothetischen weggetauscht.

## Fallback und Tripwire

**Ein zweiter Pfad, der Ausgabe erzeugt, ist unter vier Bedingungen erlaubt, und alle vier gelten gleichzeitig.**
- Er antwortet auf einen in echten Daten beobachteten Fehler.
- Er deckt diesen beobachteten Fall ab und nichts darüber hinaus.
- Das Artefakt, in dem das Ergebnis reist, benennt, welcher Pfad es erzeugt hat.
- Jede Abweichung außerhalb des beobachteten Falls scheitert laut.

**Ein zweiter Pfad, dem eine der vier Bedingungen fehlt, ist ein Fallback, und ein Fallback wird beseitigt.**

**Ein Zweig, der sich weigert Ausgabe zu erzeugen und den Fehler sichtbar macht, ist ein Tripwire, und ein Tripwire bleibt.**

**Ein Laufzeit-Fallback, der nach einem bestandenen Beweis ausgeliefert wird, misstraut dem Beweis.**
- Höchstens ein Tripwire für wirklich neuartige Eingaben bleibt übrig.

## dev/

**dev/ hält Entwicklungsskripte für Test, Debugging und Experimente.**

**Dauerhafter Wert entscheidet, was einen Platz in dev/ verdient.**
- Die entscheidende Frage ist, ob das Skript einem anderen Agenten ohne jeden Kontext nützt.
- Wenn ja, gehört es in dev/.
- Wenn nein, gehört es in den Worktree oder nach /tmp/.
- dev/ gibt einem Agenten ohne Kontext an die Hand, welche Tests liefen, wann, wie und mit welchem Ergebnis.
