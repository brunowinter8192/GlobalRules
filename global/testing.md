# Test und Verifikation

## Test

**Ein Test läuft in einer vollständig von dir kontrollierbaren Umgebung.**
- Einen Test 10 oder 100 Mal zu wiederholen ergibt jedes Mal dasselbe Ergebnis sofern du nicht aktiv die Umgebung beeinflusst.
- Einen Test in einem Monat oder in einem Jahr laufen zu lassen ergibt ebenfalls dasselbe Ergebnis sofern du nicht aktiv die Umgebung beeinflusst.
- Das ist ein hermetischer Test (Titus Winters et al., 2020) und das R für Repeatable in FIRST (Robert C. Martin, 2008).

## Wann ist ein Test erforderlich?

**Ein Test ist erforderlich, sobald sich durch eine Implementierung ein Verhalten ändert.**
- Die Anwendung zeigt aktuell Verhalten A, und die Implementierung bewegt es zu A.1 oder B.
- Der Test zeigt, dass die Spezifikation grundsätzlich verstanden wurde.
	- Der Test zeigt in einer Sandbox-Umgebung, dass du das gewünschte Verhalten zunächst mal grundsätzlich erzeugen kannst.

**Ein Meilenstein liefert eine Funktionalität, und der Test prüft genau diese ab.**
- Es werden nur Edge Cases einbezogen die auch wirklich beobachtet wurden.
	- Es wird also geprüft ob das gewünschte Verhalten auch unter einer gewissen Umweltbedingung bestehen bleibt, allerdings nur wenn diese Umweltbedingung schon einmal irgendwann beobachtbar war.
- Eine Umweltbedingung, die für dich plausibel ist aber nie vorkam, rechtfertigt keinen Test.
    - Das folgt dem Prinzip YAGNI (Kent Beck, 1999).

## Aufbau von Tests

**Unabhängige Testfälle laufen parallel.**
- Baue Tests immer so, dass eine maximale Parallelisierung erreicht ist.
- Parallele Stränge teilen keine Ports, Cache-Ordner oder Dateien.
- Rechenleistung ist bei der Parallelisierung von Tests keine Grenze, gehe von unbegrenzten Ressourcen aus.
- Das setzt das I für Independent in FIRST voraus (Robert C. Martin, 2008).

**Ein Strang bricht beim ersten Fehlschlag automatisch ab.**
- Das Fail-fast-Prinzip (Jim Shore, 2004) gilt für Tests ebenso wie für Produktionscode.
- Laufende parallele Testfälle werden nicht beendet, wenn einer der parallel ausgeführten Stränge in einen Fehler läuft.
    - Nach der Korrektur läuft nur der abgebrochene Strang erneut, und zwar einzeln.

**Die Zahl der Wiederholungen steht vor dem Lauf fest.**
- Wiederholungen zur Messung eines instabilen Tests laufen parallel.
- Ihre Zahl wird einmal festgelegt und begründet, nicht schrittweise erhöht.
    - Sollte sich herausstellen, dass weitere Wiederholungen notwendig sind, erhöhe nie einfach so das Kontingent.
    - Benachrichtige im Chat den User oder den Main Agent und gehe sofort idle.
- Schrittweises Erhöhen bis zum gewünschten Ergebnis ist das Problem Optional Stopping (Joseph Simmons, Leif Nelson, Uri Simonsohn, 2011).

## Verifikation

**Eine Verifikation entspricht genau der Produktionsumgebung.**
- Die Verifikation läuft, wenn Tests keinen Erkenntnisgewinn mehr bringen.
- Wirkt eine Änderung erst nach einer Veröffentlichung, zum Beispiel per `plugin-publish`, wird vor der Verifikation veröffentlicht.
    - Zur Not wird die Veröffentlichung danach zurückgerollt.

**Eine Verifikation, die das beabsichtigte Verhalten A.1 oder B nicht zeigt, triggert neue Tests.**
- Es wird zunächst die Implementierung geändert, dann erneut getestet, dann erneut verifiziert.
- Scheitert eine Verifikation zweimal, so werden alle Aktionen sofort gestoppt und du reportest im Chat.
	- Was wurde getestet.
	- Welche Symptome wurden beobachtet.
	- Welche Hypothesen drängen sich auf die das beobachtete Verhalten erklären.

## Fallback und Tripwire

**Ein Fallback ist ein zweiter Weg zum selben Verhalten.**
- Bei mehreren Wegen entscheiden die Umweltbedingungen welcher zum Verhalten führt.
- Um einen Fallback zu realisieren muss die entsprechende Umweltbedingung beobachtet worden sein.
- Plausible jedoch nicht beobachtete Umweltbedingungen sind kein Anlass für einen Fallback.
    - Das folgt dem Prinzip YAGNI (Kent Beck, 1999).

**Ist ein Fallback realisiert, so muss jeder Weg extern nachvollziehbar sein.**
- Zum Beispiel durch Logging kann sichergestellt werden, dass immer klar ist welcher Weg das Verhalten erzeugte.

**Ein Tripwire fängt jede Umweltbedingung ab, die nicht beobachtet wurde.**
- Der Tripwire unterscheidet nicht zwischen diesen Bedingungen, sondern wirft sie alle in einen Topf.
- Es braucht deshalb nur einen Tripwire, egal wie viele Bedingungen denkbar sind.
- Der Tripwire folgt dem Prinzip Fail Fast (Jim Shore, 2004).

**Die Reaktion eines Tripwires ist immer dieselbe: abbrechen und den Fehler melden.**
- Der Tripwire erzeugt nie das Ergebnis eines ordnungsgemäßen Durchlaufs, sondern nur einen Fehler, deshalb braucht er keinen Verhaltensbeweis.
- Welche Bedingung den Tripwire ausgelöst hat, klärt die Untersuchung im Nachgang über Logs.
