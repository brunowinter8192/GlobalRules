# Konvention für das dev/-Verzeichnis

**In welchem dev/-Ordner du arbeitest richtet sich ausschließlich nach der Area, in der sich die Session bewegt.**
- Unterordner von dev/ sind exakt so benannt wie die zugehörige Area.
- Ein dev/-Ordner hat immer eine zugehörige Area, während eine Area nicht zwangsläufig einen zugehörigen dev/-Ordner haben muss.

**dev/ hält Entwicklungsskripte für Experimente aller Art.**
- In dev/ kannst du dich grundsätzlich frei entfalten, es gibt nur wenige Restriktionen:
    - Alles was in dev/ liegt ist persistent, Inhalte die ein folgender Agent also unter keinen Umständen brauchen kann, gehören nach /tmp/.
    - Inhalt in Wort und Schrift gehört in die process-docs, ausgenommen ist der von einem Skript erzeugte Report.

**Ein dev-Skript welches einen Report erzeugt, muss den Report in einem dafür vorgesehenen Unterverzeichnis ablegen.**
- Es können beliebige Ordner wie `md/`, `csv/` oder `png/` angelegt werden, abhängig vom Format des Outputs.
- Das Unterverzeichnis liegt immer auf derselben Ebene wie das Skript, das den Report erzeugt.
- Die Report-Datei trägt einen Namen, der auf das erzeugende Skript zurückführt.
    - Es muss zweifelsfrei identifizierbar sein, welches Skript welchen Report erzeugt hat.
