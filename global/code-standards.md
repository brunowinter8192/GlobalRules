# Code-Standards

## Kernregeln

**Die Ordner debug/ und logs/ sind kein Teil der Version Control.**
- Beide Ordner müssen zudem in .gitignore stehen.

**Emojis landen nicht in Produktionscode, READMEs, DOCS.md und Logs.**

## Modulaufbau

**Die Sektionen eines Moduls stehen immer in der Reihenfolge INFRASTRUCTURE, ORCHESTRATOR, FUNCTIONS.**
- Eine Sektion, die leer bliebe, wird weggelassen.
- Ein Modul, das mehr als einen Arbeitsschritt ausführt, hat immer einen ORCHESTRATOR.

### INFRASTRUCTURE

**INFRASTRUCTURE beinhaltet die Imports und die Konstanten des Moduls.**
- Konstanten, die nur von diesem einen Modul gebraucht werden, tauchen in dieser Sektion auf.
- Konstanten, die sich zwei oder mehr Module teilen, tauchen im config-Modul auf und werden von dort importiert.

### ORCHESTRATOR

**Der ORCHESTRATOR ist genau eine Funktion, und diese Funktion ruft ausschließlich andere Funktionen auf.**
- Der Orchestrator ist die Imperative Shell aus dem Muster Functional Core, Imperative Shell.
- Der Orchestrator ist zugleich ein Humble Object, er wird also so dumm gehalten, dass er keinen eigenen Test braucht.
- Bei einem CLI Command trägt der Orchestrator den Namen `<command>_workflow`, ansonsten ist der Name frei wählbar.
- Der Orchestrator selbst enthält keine funktionale Logik.
   - Er darf über eine Bedingung entscheiden, welche Funktion als nächstes aufgerufen wird.
   - Er darf das Ergebnis einer Funktion an die nächste Funktion weitergeben.

### FUNCTIONS

**Die FUNCTIONS stehen in der Reihenfolge, in der sie aufgerufen werden.**
- Die FUNCTIONS sind der Functional Core, die gesamte funktionale Logik des Moduls steht in dieser Sektion.
- Die Reihenfolge der Funktionen folgt der Stepdown Rule, der Leser kommt von oben nach unten immer tiefer ins Detail.
- Jede einzelne Funktion hat genau eine Verantwortung, ganz im Sinne des Single Responsibility Principle.
- Funktionen dürfen sich gegenseitig aufrufen.
- Jede Funktion ist vom Orchestrator aus erreichbar, entweder direkt oder über eine andere Funktion.

## Kommentare und Docstrings

**Kommentare und Docstrings sind im Code verboten.**
- Jegliche Arten von Kommentaren und Docstrings sind nicht erwünscht:
   - Docstring an einem Modul, einer Klasse oder einer Funktion.
   - Überschrift über einem `def`.
   - Trailing Comment an einer Anweisung.
   - Anmerkung an einem Import.
- Alles was du als Erklärung zu einem Modul erfassen möchtest, kommt in die process-docs.

**Die drei Section Marker `# INFRASTRUCTURE`, `# ORCHESTRATOR` und `# FUNCTIONS` sind die einzigen erlaubten Kommentarzeilen.**

## Import-Konvention

**Ein Import wird absolut geschrieben, so wie PEP 8 es empfiehlt.**
- Die Form ist `from src.module.submodule import name`.

## Abhängigkeiten zwischen Modulen

**Braucht Modul A Funktionalität aus Modul B, dann importiert Modul A die benötigten Funktionen aus Modul B.**
- Der Orchestrator von Modul A ruft diese importierten Funktionen auf.
- Eine Funktion, die ausschließlich von einem anderen Modul genutzt wird, gehört in dieses andere Modul.
