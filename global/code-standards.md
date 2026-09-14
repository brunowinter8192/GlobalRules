# Code-Standards

## Kernregeln

- Die Ordner debug/ und logs/ sind kein Teil der Versionskontrolle und müssen zudem in .gitignore stehen.
- Emojis landen nicht in Produktionscode, READMEs, DOCS.md und Logs.

## Modulaufbau

**Jedes Modul folgt INFRASTRUCTURE, dann ORCHESTRATOR, dann FUNCTIONS.**

| Abschnitt | Inhalt |
|---|---|
| INFRASTRUCTURE | Imports und Konstanten. Funktionen und Logik gehören nicht hierher. Modulspezifische Konstanten leben hier. Konstanten, die zwei oder mehr Module teilen, gehören ins config-Modul und werden importiert. |
| ORCHESTRATOR | Eine Funktion, benannt `<command>_workflow` bei CLI-Kommandos und ansonsten frei benannt. Sie ruft nur andere Funktionen auf und enthält null funktionale Logik. Bedingte Workflow-Ausführung und Parameter-Routing sind erlaubt. |
| FUNCTIONS | Geordnet nach Aufrufreihenfolge, je eine Verantwortung. Funktionen dürfen intern andere Funktionen aufrufen. Jede Funktion ist vom Orchestrator aus erreichbar, direkt oder indirekt. |

**Utility-Module sind die Ausnahme.**
- Module, die nur Konstanten enthalten, client.py und Helper dürfen die Abschnitte ORCHESTRATOR und FUNCTIONS weglassen.

## Kommentarregeln

**Die einzigen Kommentarzeilen sind die drei Abschnittsmarker.**
- `# INFRASTRUCTURE`, `# ORCHESTRATOR`, `# FUNCTIONS`.

**Jeder andere Kommentar ist verboten.**
- Kein Docstring an einem Modul, einer Klasse oder einer Funktion.
- Keine Überschrift über einem `def`.
- Kein nachgestellter Erklärer an einer Anweisung.
- Keine Anmerkung an einem Import.

## Import-Konvention

**Absolute Imports bevorzugen.**
- Die Form ist `from src.module.submodule import name`.
- Relative Imports bleiben aus Projekten heraus, die absolute Imports durchgängig verwenden.

## Abhängigkeiten zwischen Modulen

**Wenn Modul A Funktionalität aus Modul B braucht, importiert A bestimmte Funktionen aus B.**
- Der Orchestrator von Modul A ruft die importierten Funktionen auf.
- Eine Funktion, die nur ein anderes Modul nutzt, gehört in dieses Modul.

## Namenskonventionen

| Element | Konvention |
|---|---|
| Domänenordner | `src/domain_name/`, snake_case und beschreibend |
| Module | `src/domain/module_name.py`, snake_case |
| Paketmarker | `src/__init__.py` und `src/domain/__init__.py`, für Imports erforderlich |
| Dokumentation | `src/domain/DOCS.md`, eine pro Domäne |
