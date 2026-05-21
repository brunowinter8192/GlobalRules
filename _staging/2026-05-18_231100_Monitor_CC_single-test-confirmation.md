# 2026-05-18 — Monitor_CC: single-test confirmation premature

## ~/.claude/shared-rules/global/verify-before-execution.md → Correlation Check Before Root-Cause Claim

**Problem:** Bei der PDF-Focus-Bug-Diagnose schrieb Opus nach EINEM User-Test ("unload menubar + 1× MissionControl-Klick = Bug weg") sofort "Bestätigt — Menübar war's." 15 Minuten später meldete der User: Bug existiert auch ohne Menübar.

**How it should be:** Single positive diagnostic test = TENTATIVE evidence, never CONFIRMATION.

**Rule:** NEVER use confirmation-language ("bestätigt" / "confirmed" / "ist es" / "war's") after ONE diagnostic data point. Require either (a) ≥2 independent tests with consistent result, or (b) user explicitly states "I tested N≥2 times, consistent". Until the bar is met: use "indication" / "suggests" / "first test points to X — bitte 2-3× hintereinander testen".

**Concrete example:** 2026-05-18 PDF-Focus-Investigation. After user said "jap jetzt klappt es" on first test post-`launchctl unload`, Opus wrote "Bestätigt — Menübar war's." User later did more careful testing (Desktop-1→Desktop-2-Wechsel + zu Ghostty + zurück zur PDF) und der Bug trat auf trotz unloaded Menübar. Erste positive Beobachtung war false-positive — MissionControl-Click klappt manchmal beim ersten Mal unabhängig vom Focus-Stealer-State. Hätte heißen müssen: "Erster Test zeigt Bug weg — bitte 2-3× hintereinander, mit unterschiedlichen App-Konstellationen." Bevor dann erst "bestätigt".
