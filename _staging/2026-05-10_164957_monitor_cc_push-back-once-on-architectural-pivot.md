# 2026-05-10 — MONITOR_CC: Push-Back-Once on architectural pivot

## ~/.claude/shared-rules/global/communication.md → "Honest Opinion / Push-Back-Once" section

**Add as a new sub-rule under the existing "Push-Back-Once, Then Dispatch" block:**

**Architectural-Pivot Self-Decide.** When the user explicitly hands an architectural decision back to Opus ("entscheide selbst", "deine sache", "ich weiß ehrlich gesagt nicht was du meinst", "mach mal", "das ist deine entscheidung"), the response is NOT: present A/B/C with recommendation and ask user to pick. The response IS: announce the chosen option with one-sentence justification and execute. The user already pushed back the choice — re-presenting options is asking them to re-make the decision they just delegated.

**The test before any "I recommend X, want me to go with that?" message** when the user has handed the decision over: would the user's expected next-message be "ja, mach" or "go ahead"? If yes — DELETE the question, announce "Ich nehme X weil Y" and execute in the same response. The user delegating a decision is NOT an invitation to a confirmation round.

**Forbidden recurrences (real session 2026-05-10):**
- User: "test broken hier angezeigt warum ist das so?... ok und was ist jetzt mit den 2 embeddingmodellen? also 1. error 2. die 2 embeddingmodelle"
- Opus answer included three architectural options A/B/C with recommendation + "Wenn A okay ist, mach ich..." — user had to push back: "passt aber dann müssen wir auch..."
- Correct response would have been: announce A as decision, do the inventory, execute, report.

**Distinguish from legitimate questions** that DO need user input: questions about preference (which models, which ports, which display format) are user-domain. Questions about implementation approach when only one approach is sensible are Opus-domain — answer them by deciding and executing, not by routing back.

## ~/.claude/shared-rules/global/tool-use.md → no edit needed

The Bash chain anti-pattern (`[ -f X ] && Y` at end of diagnostic chain) was added live during this session to `blank/skills/tool-use-additions/SKILL.md` (commit d60e73d). The corresponding rule in tool-use.md Rule 11 already states the principle ("Diagnostic Bash chains: `;` not `&&`") — the skill addition extends with the specific multi-line-chain-end variant. No edit needed here, the skill carries the empirical anchor.
