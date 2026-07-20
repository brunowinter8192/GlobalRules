# Verification — Say Exactly What You Tested and How Far It's Proven

**Never flag something as functionally verified beyond what you actually ran.**
A pure-function assert is not proof that the production path works; a passing probe on a copy is not proof the real code works. On every task, state explicitly HOW you tested, up to WHICH point it is verified, and WHAT you conclude from it. That transparency matters more than hitting any prescribed test shape.

## Verification Levels

Descriptive, not a mandatory order — reach the levels the change actually needs, and name which you reached.

| Level | Proves | Does NOT prove |
|---|---|---|
| Pure-function regression guard | the function still returns what it did before — refactor protection | that it meets the real contract; a consistency check, not a functional proof |
| Integration test | the real function, called with real inputs → real I/O → asserted outcome | that the user-facing routing / arguments / auto-detect around it work |
| Entry-point (CLI / HTTP) | the real caller path end-to-end — routing, arguments, auto-detect included | visual / rendered correctness a human must judge |
| User visual check | rendered output is right, judged by a human | nothing above it — the last gate, used only when self-verification is impossible |

If you only imported and called the function, say so plainly: "verified at integration level, NOT at the entry-point." The honest boundary is the deliverable — not a green checkmark that overstates it.

## Report Honestly

- **State the verified boundary — before being asked.**
  Spell out exactly how you tested and up to which level it holds; flag any part only code-reviewed and not run, up front — not after the user or Opus asks.
- **Separate regression guards from integration tests.**
  In the checklist, split pure-function asserts from real call → real I/O → assert tests. Never just "X/X passing".
- **PARTIAL is a legitimate status.**
  Planned verification blocked by an unrelated cause (CAPTCHA, 503, missing data) → report PARTIAL and say precisely what ran, what did not, and why. Never inflate PARTIAL into "verified" with a weaker substitute that does not test the contract. Opus picks it up and decides the next step.

Example reporting format:

```
Tests: 6 integration (passing) + 52 regression-guards (passing).
Verified: URL-transform end-to-end via CLI → real output on disk; blacklist + error path.
NOT verified: visual rendering of the report — needs a user check.
```

## Pre-Commit Checks

Before the final commit, run the mechanical checks the change actually needs — not a mandatory all-five sweep:

1. **Syntactically valid.** `python -c "import ast; ast.parse(open('path').read())"`
2. **Imports resolve.** Every imported module/function exists in the codebase.
3. **Library methods exist.** For external classes, verify the methods you call: `python -c "from lib import Class; print([m for m in dir(Class()) if not m.startswith('_')])"`. Never trust training data for method names.
4. **Pattern compliance.** File structure matches the reference file — same sections, same style.
5. **Edge cases.** Prompt-named data formats (URNs, URLs, timestamps) — verify your parsing handles them.
