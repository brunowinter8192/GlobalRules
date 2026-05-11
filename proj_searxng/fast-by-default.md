# Fast by Default — No Defensive Timing

Dieses Projekt hat eine Geschichte von defensiv-konservativen Timings die sich aufaddieren: Polling-Loops mit 15× 1-Sekunden-Sleeps, 2s Consent-Sleeps "zur Sicherheit", Rate-Limiter pessimistisch auf 4 req/min weil "vielleicht hat der Server nen Bot-Filter", 5min-Cooldowns zwischen Test-Runs "damit der Server sich erholt". Resultat: Smoke-Tests dauern 8min statt 2, Single-Query-Suchen 12s statt 3, Scholar blockiert die ganze User-Query 14s wenn die Page mal 0.5s langsamer rendert.

**Regel: alles as fast as possible. Defensive Pausen sind verboten — Errors werden gemeldet, nicht weggewartet.**

## Was das konkret bedeutet

### In `src/` (Produktionscode)

- **Polling-Loops:** maximal 3 Zyklen mit 0.1-0.3s Intervall. Wenn das Resultat nicht in 0.3-0.9s da ist, ist es entweder ein echter Error (sofort raisen) oder hängt (Worst-Case-Timeout greift, Caller informieren).
- **`asyncio.sleep` außerhalb von Rate-Limitern:** verboten. Jedes `sleep()` in Engine-Code, Workflow-Code, Scraper-Code muss begründet sein durch eine konkrete API-Doc-Anforderung (z.B. "OpenAlex docs say wait 1s after 429"). "Damit der DOM settled" / "damit Consent-Banner schliesst" sind keine gültigen Gründe — bestätige per JS-Wait dass das Element da ist, oder fail.
- **Rate-Limiter-Werte:** orientieren sich am tatsächlichen API-Limit oder anti-bot-Threshold, NICHT an "wir sind höflich". HTTP-API-Engines mit dokumentierten Limits (OpenAlex 100k/day, CrossRef 50/s polite-pool, Stack Exchange 30/s) bekommen mindestens diese Werte. Browser-Engines bleiben konservativer wegen real existierender Bot-Filter, aber niemals defensiver als empirisch nötig (Probe → Burst-Test → Rate setzen).
- **Timeouts:** sized to actual empirisch maximum observed — KEIN Multiplier, KEINE "Sicherheits-Marge", KEIN "× 1.5 weil safe". Wenn die Probe maximum 4.6s misst, ist der Timeout 4.6s (oder die nächste runde Zahl die nicht drunter liegt: 5s). Wenn die Realität später einmal 5.2s braucht → der eine Request stirbt mit Timeout, der User sieht "1 von 8 engines failed", die anderen 7 liefern weiter ihre Results. Das ist akzeptable Failure-Rate. Multiplier-Denken ("× 1.5 falls mal was länger dauert") ist die Falle — die kostet bei JEDEM Request Zeit, nicht nur bei den seltenen Ausreissern.

  **Die Asymmetrie verstehen:** Cost of occasional fail = 1 Request kriegt Error, bounded. Cost of always-being-slow = jede einzige User-Query verschwendet x% Zeit, accumulates über alle Requests. Letzteres ist schlimmer. Empirie ist der Wert, nicht "Empirie × Buffer".

### In `dev/` (Probes, Smokes, Evals)

- **Cooldowns zwischen Runs:** verboten ausser bei explizitem Anti-Rate-Limit-Bedarf den die Probe-Befunde dokumentieren. Standard-Annahme: kein Cooldown nötig, sequenzielle Runs ok.
- **`time.sleep` / `asyncio.sleep` zwischen Operations:** verboten ausser pro-Domain-Höflichkeit (max 0.5s) wenn die Probe gegen denselben Host viele URLs probt. Globale Sleeps "damit nichts crasht" sind Bullshit.
- **Polling-Loops in Smokes:** bei script-internem Warten (z.B. auf eine Page) gilt das gleiche wie in src/ — 3×0.2s, dann fail.
- **Per-Query-Wait in Smokes:** Wenn der Smoke 30 Queries hintereinander fährt, brauchts keinen Sleep zwischen Queries — die Engine selbst hat ihren Rate-Limiter, der reicht.

### In Worker-Sessions (orchestriert von Opus)

- **`Bash(command="sleep N")` in Workern:** verboten — siehe `worker-script-execution.md`. Skripte laufen im Vordergrund mit `timeout=600000`, fertig.
- **Defensive Wait-Pattern wie "5min nach Skript-Start checken":** verboten. Skript läuft, Skript blockt den Bash-Call, Skript returnt, Worker liest Output. Kein Polling.

## Uniform-Across-Engines (or Equivalent Parallel Resources)

Im searxng-Workflow laufen 8 Engines parallel via `asyncio.gather`. Die Wall-Clock-Zeit pro Query ist `max(engine_zeiten)` — der langsamste Engine bestimmt das Tempo, alle anderen warten ohnehin. Daraus folgt **deterministisch**: engine-spezifische Timeouts unterhalb der langsamsten-Engine-Effective-Max sind nutzlos. Sie killen einzelne Engines vorzeitig ohne irgendeine Wall-Clock-Ersparnis.

**Regel:** alle Engines die im selben gather-Pfad laufen, bekommen den gleichen Timeout pro Layer. Per-Engine-Asymmetrie ist die Default-Verdacht-Position — sie braucht eine konkrete empirische Begründung um zugelassen zu werden. "Engine X ist schneller, also tighter Timeout" ist **keine** Begründung — der gather wartet trotzdem auf den langsamsten, X kann genauso lange Zeit kriegen ohne Verlust.

Was als legitime engine-spezifische Begründung zählt:
- Engine hat Layer Y nicht (z.B. HTTP-Engines haben kein "page-load")
- Engine's Anti-Bot-Threshold ist nachweislich strikter als die Mehrheit (Probe mit 429-Triggers)
- Engine's API hat ein dokumentiertes Hard-Limit das niedriger ist als die anderen

**Konkretes Beispiel:** Browser-Engines können effektiv 3.6s brauchen (page-load 3s + polling 0.6s). HTTP-Engines bekommen den GLEICHEN 3.6s-Maßstab via `httpx.Timeout(3.6)` — auch wenn HTTP empirisch im Median schneller ist. Würde man HTTP auf 1.5s setzen, würde ein langsamer aber legitim-funktionierender HTTP-Call gekillt während ein Browser-Call ähnlicher Latenz nicht. Pure Asymmetrie ohne Wall-Clock-Gewinn.

**Failure-Pattern aus Session 2026-05-07:** drei Iterationen im selben Fehler. Opus hat zuerst max-observed × 1.5 Multiplier vorgeschlagen, dann HTTP=1.5s während Browser=3s, dann Watchdog=3s während Browser intern 3.6s konnte. Jedes Mal User-Korrektur mit demselben Argument: "asyncio.gather wartet auf den langsamsten, dein per-Engine-Tighter ist sinnlos". Diese Regel kodifiziert das Argument permanent — wenn du das nächste Mal in Versuchung kommst eine engine-spezifische tightere Konstante vorzuschlagen, lies diese Sektion erneut.

## Lock-ins sind Lock-ins

Wenn ein Wert basiert auf einer empirischen Probe gelockt-in wird, bleibt er gelockt. Nicht später "× 1.5 buffer weil safe". Nicht später "vielleicht 5s damit nichts hängt". Nicht später "kurze Variation testen". Der Wert ist fixiert sobald die Probe-Evidenz da ist.

Ausnahmen die einen neuen Lock-in rechtfertigen:
- Neue Probe mit echten Daten zeigt dass der Wert nicht ausreicht (z.B. dauerhafte TIMEOUT-Rate >5% bei einer Engine)
- Externe Constraint ändert sich nachweisbar (Engine ändert Anti-Bot-Threshold, neue API-Version mit anderem Latenz-Profil)

In beiden Fällen: neuer Probe-Run, neue Daten, neuer Lock-in mit der GLEICHEN Disziplin (max-observed = der Wert, kein Multiplier). Buffer-Adding ohne empirische Evidenz ist exact die Falle die wir vermeiden.

## Fail-Fast-Prinzip

Errors müssen sofort sichtbar werden — nicht hinter Retry-Loops und Exponential-Backoff verschwinden.

- **Try/except mit silent pass:** verboten. Jede gefangene Exception muss entweder geloggt werden (mit Kontext was schief ging) oder propagiert werden.
- **Exponential-Backoff bei API-Errors:** nur bei API-spezifischen 429/503-Rückmeldungen, mit dokumentiertem Maximum (max 3 Versuche, max 8s gesamt). Niemals bei generischen Connection-Errors — die signalisieren echte Probleme die der User wissen muss.
- **`functools.lru_cache` / Result-Caching ohne TTL:** verboten in src/. Cache-Inkonsistenz versteckt Bugs; explizite Cache-Strategie (sha256-key + TTL wie in src/search/cache.py) ist die Alternative.
- **Hänger:** wenn ein Skript / Engine / Workflow >10s ohne Output ist, gehört das in die Logs als WARNING. Wenn Opus oder ein Worker das beobachtet, MUSS es dem User berichtet werden — nicht stillschweigend abgewartet.

## Kontext

Bead `searxng-8dg` (Timing-Migration) ist die konkrete Erstanwendung dieser Regel — drei Konstanten-Klassen in src/search/engines/ migrieren von defensiv auf empirisch-validiert-aggressiv. Die Regel hier kodifiziert das Prinzip damit zukünftige Code-Additionen nicht wieder defensives Timing einschleppen.

Was diese Regel NICHT sagt:
- Sie sagt nicht "ignoriert Rate-Limits" — sie sagt "respektiert die ECHTEN Limits, nicht eure Phantasie-Limits"
- Sie sagt nicht "Skript ohne Error-Handling" — sie sagt "Error-Handling muss Errors SICHTBAR machen, nicht maskieren"
- Sie sagt nicht "alle Sleeps weg" — sie sagt "jeder Sleep braucht eine konkrete dokumentierte Begründung"

## Mitigation-Choice Discipline (NON-NEGOTIABLE)

Wenn ein Production-Bug einen Fix verlangt UND mehrere Optionen auf dem Tisch liegen, gilt eine harte Hierarchie der akzeptablen Wallclock-Auswirkungen:

**Rang 1 — Absorbiert (vorzuziehen):** der Fix passt in eine bereits existierende Wartezeit der Pipeline. Beispiel: Engine A fällt aus weil sie mit Engine B kollidiert. Wenn ein 1.5s Pre-Delay vor A's Request den Konflikt auflöst UND die langsamste parallel laufende Engine sowieso 5s braucht (weil sie eh Server-dominated ist), dann ist A's Pre-Delay unsichtbar im Wallclock — `max(other_engines, A_with_delay) = max(other_engines)` solange `A_with_delay ≤ slowest_other`. Sichtbarer Cost: 0ms.

**Rang 2 — Visible (nur mit explizitem User-OK):** der Fix verlängert die Wallclock pro Query messbar. Beispiel: Engine A sequenziell nach `asyncio.gather` laufen lassen statt drin → `total = max(other_engines) + A_time` statt nur `max(other_engines)`. Auch wenn A_time klein ist (~700ms), ist das ein deterministischer Aufschlag auf JEDE relevante Query. Diese Klasse wird nur angenommen wenn (a) Rang 1 empirisch nicht funktioniert hat ODER (b) der User die Wallclock-Cost explizit akzeptiert.

**Rang 3 — Coverage-Drop (sofort wenn null Wallclock-Cost null akzeptabel):** Engine A wird in der betroffenen Konfiguration aus dem Set genommen. Wallclock unverändert, aber Result-Coverage fehlt. Akzeptabel wenn die Engine in dem Modus eh nicht funktioniert (also Coverage faktisch eh schon null ist).

**Rang verboten — "ein bisschen langsamer ist OK":** kein Vorschlag der Wallclock pro Query ohne empirischen Need verlängert ist verhandelbar. Die Default-Annahme ist: Rang 1 oder Rang 3, niemals Rang 2 ohne explizite Begründung warum Rang 1 ausgeschlossen ist.

**Test vor jeder Mitigation-Empfehlung:** rechne die Wallclock-Mathematik per Rang aus, in dieser Reihenfolge:
1. Gibt es einen Rang-1-Pfad der absorbiert? Wenn ja → empfiehl ihn.
2. Wenn nein, ist die Engine im betroffenen Modus eh non-functional? Wenn ja → Rang 3 (drop).
3. Wenn beides nein → erst hier kommt Rang 2 in Frage, MIT expliziter Wallclock-Cost-Frage an den User.

**Konkreter Fall (2026-05-08):** Bead searxng-f3i Step 2 (Scholar-Google-Decoupling). Optionen:
- (a) Scholar sequenziell nach gather → +700ms unconditional auf academic-Queries mit Google = Rang 2
- (b) Scholar pre-acquire delay 1.5s, conditional auf Google im Set → absorbiert in slowest-engine-Window (open_library 6s, semantic_scholar 5s, crossref 6s) = Rang 1
- (c) Scholar dropped wenn Google im Set → Rang 3, aber Scholar wäre dann komplett aus default --academic queries weg, Coverage-Cost real

Korrekte Empfehlung: (b). Ich (Opus) hatte initial (a) empfohlen — Verstoß gegen diese Regel, vom User korrigiert. Diese Regel verhindert die Wiederholung.

## Concrete failure (warum diese Regel jetzt entsteht)

Session 2026-05-07. Worker `jqn-foundation` brauchte ~12 min für eine 3-File-Implementierung mit Verifikation. Davon:
- 8+ Background-Sleeps (60s, 120s, 180s) für Smoke-Wartezeiten — Pattern aus Opus-Orchestrierung in Worker geleakt (siehe worker-script-execution.md)
- Status-Quo Scholar-Polling 15×1s in den Verifikations-Runs — wenn ein Smoke 30 Queries macht und Scholar's Polling jedes Mal die volle Page-Load-Zeit ausreizt, kommen pro Query 1-2s aufaddiert vom Scholar-Tab dazu, multipliziert mit 30 = 30-60s Overhead
- Smoke-Skripte mit defensiven Cooldowns die jede Test-Iteration auf 8min strecken obwohl die eigentliche Arbeit 2min dauert

Resultat: User hat zugesehen wie Worker 12 Minuten braucht für Code der in 3 Minuten erledigt sein sollte. Bead `searxng-8dg` adressiert die konkreten Konstanten in src/, diese Regel adressiert das Mindset für alle zukünftigen Additionen.
