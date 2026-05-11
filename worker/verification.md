# Verification — Test the Real Path, Not a Parallel Reimplementation

Tests und Verifications sind nur etwas wert wenn sie das ECHTE Verhalten prüfen das ein User trifft. "X/X passing" sagt nichts wenn X Tests die falsche Sache prüfen oder triviale Regressions-Guards sind.

## Die vier Failure-Patterns die diese Regel adressiert

### Pattern 1 — Wrong Path: Tests prüfen ein paralleles Reimplement, nicht die Production-Funktion

Wenn der Worker eine Funktion in `src/foo.py` baut und ein dev-Skript ähnliche Logik inline duplicated für eine Probe, ist die Probe **kein Regressions-Check** für die `src/` Funktion. Sie testet den parallelen Code-Pfad.

**Konkret diese Session 2026-05-07:** PDF Chain wurde nach `src/scraper/pdf_chain.py` migriert. "Regression check via probe 16" wurde als Verifikation deklariert. Probe 16 hatte aber seine **eigene inline Chain-Logik** (nicht die migrierte). Probe 16 bestand → Worker meldete "10 PDFs in Downloads, baseline OK, migration done". Live-Test 30 Min später: `searxng-cli download_pdf https://arxiv.org/abs/X` → Fehler "no PDF path found". Bug war in `download_pdf_workflow` (production path), Probe hatte ihn nicht getroffen weil Probe ihre eigene Chain hatte.

**Regel:** wenn du eine Funktion in `src/` migrierst oder änderst, muss mindestens **eine** Verification die echte `src/`-Funktion aufrufen — direkt importieren, mit echten Inputs, mit assertion auf das echte Output. Nicht eine Funktion mit gleichem Namen in einem Probe-Skript. Nicht eine Reimplementierung die "äquivalent sein sollte". Die echte.

### Pattern 2 — Trivial Asserts gegen pure Funktionen als "Tests" gezählt

```python
def test_arxiv_abs(self):
    assert apply_tier1_transform("https://arxiv.org/abs/2501.12345") == "https://arxiv.org/pdf/2501.12345"
```

Das ist kein Test im Sinne von "validiert ein Contract gegen die Realität". Das ist ein Code-Konsistenz-Check: "die Funktion gibt zurück was wir reingeschrieben haben". Hat Wert wenn jemand später die Konstante / Regex ändert ohne nachzudenken — fungiert als Refactor-Schutz. Hat **keinen** Wert als Funktional-Beweis.

**Regel:** wenn du Tests in einer Completion-Checklist meldest, trenne **regression guards** (pure-function asserts ohne Contract-Validation) von **integration tests** (call → real I/O → assert auf Outcome). Beispiel-Format:

```
Tests: 6 integration (passing) + 52 regression-guards (passing).
Integration: arxiv abs/html/version-suffix end-to-end download_pdf_workflow → real PDF on disk;
            blacklist + github-blob error path; openreview /forum?id= chain.
Regression-guards: pure-function asserts on transforms/blacklist/regex behavior.
```

Niemals nur "X/X passing" ohne diese Trennung. Der User soll auf einen Blick sehen was wirklich verifiziert wurde vs was Boilerplate ist.

### Pattern 3 — PARTIAL ohne Fallback-Verification

Wenn die geplante Verification durch eine **unrelated** Ursache scheitert (Scholar CAPTCHA hängt, Server 503, Test-Daten fehlen), ist die Antwort NICHT "ich melde PARTIAL und stoppe". Die Antwort ist: finde eine kleinere oder andere Verification die das Contract trotzdem trifft.

**Konkret diese Session 2026-05-07:** jqn-Worker plante `11_pipeline_smoke.py --max-queries 5` als Verifikation. Q2+ blockierte durch pre-existing Scholar-CAPTCHA-hang. Worker meldete "PARTIAL — pre-existing issue, nicht durch jqn changes verursacht". Was er hätte machen sollen: 1-Query-Smoke, oder `01_google_smoke.py --queries 3` (umgeht Scholar), oder direkter `search_web_workflow`-Call mit `engines="google,duckduckgo"`. Die jqn-Änderungen waren in 4 verschiedenen Pfaden (selector, max_results, modifier hook) — jeder einzeln testbar ohne Scholar zu involvieren.

**Regel:** "PARTIAL" als Verifikations-Status ist akzeptabel nur wenn der Worker mindestens 2 alternative kleinere Verifications versucht hat und im Report listet warum jede gescheitert ist. Der Default ist: zerlege die Verifikation in kleinere Teile die einzeln gehen, statt aufzugeben weil das grosse Smoke nicht durchläuft.

### Pattern 4 — User-visible Entry Point überspringen

Code der einen CLI/MCP/HTTP-Entry-Point hat MUSS mindestens einmal über diesen Entry-Point verifiziert werden. Direkter Python-Import + Funktions-Call ist NICHT genug — der CLI-Wrapper-Pfad hat oft eigene Routing-/Argument-/Auto-Detect-Logik die ein direkter Import umgeht.

**Konkret diese Session 2026-05-07:** `download_pdf_workflow` wurde via Python-Import in probe 16 getestet. Der eigentliche User-Pfad ist aber `searxng-cli download_pdf <url>` — das geht durch `cli.py` Auto-Routing inklusive `should_download_as_pdf()`-Check. Der `cli.py`-Pfad wurde nicht verifiziert. Live-Test über CLI hat den Bug zuerst gezeigt.

**Regel:** in der Completion-Checklist muss eine Zeile stehen die explizit den User-facing Entry-Point invokiert: ein `searxng-cli ...` Aufruf, ein MCP-tool-call, ein curl gegen den HTTP-Endpoint. Nicht bloss Python-Import. Output dieses Aufrufs gehört in die Checklist (gekürzt).

## Was Verification "done right" aussieht

In der Completion-Checklist eines Workers gehören in dieser Reihenfolge:

1. **Pure-function regression guards** — kurz benannt mit Anzahl, klar als solche markiert. Beispiel: "Regression-guards: 52 pure-function asserts (transforms, blacklist, regex behavior). Passing."
2. **Integration tests gegen die echte src/-Funktion** — mindestens einer pro neuer/geänderter Funktion. Mit konkretem Input und konkretem Outcome-Assert.
3. **End-to-end Verification über den User-facing Entry-Point** — CLI-Aufruf, MCP-tool, oder HTTP-Request. Mit echtem Output gekürzt.
4. **Sonstige Verifications** (Smoke-Runs, Sample-Tests) wenn relevant.

Wenn ein Schritt aus 1-3 nicht möglich ist → expliziter Eintrag in der Checklist warum, plus Alternative die statt dessen das Contract trifft.

## Was diese Regel NICHT sagt

- Sie sagt nicht "schreibt keine Unit-Tests" — schreibt sie, aber meldet sie als das was sie sind (Regression-Guards), nicht als "Verification".
- Sie sagt nicht "jeder Test muss network-I/O machen" — pure-function tests haben Wert als Refactor-Schutz, sie sind nur kein Funktional-Beweis.
- Sie sagt nicht "End-to-end-CLI-Test kann nie skipped werden" — wenn der Worker keinen CLI/MCP-Endpoint geändert hat (z.B. nur Engine-Logik refactored), reicht Integration-Test ohne CLI-Roundtrip. Aber der Test muss die Funktion über den Pfad erreichen den ein realer Caller benutzen würde.
