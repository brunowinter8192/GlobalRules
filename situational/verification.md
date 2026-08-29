# Verification — Say Exactly What You Tested and How Far It's Proven

**Never flag something as functionally verified beyond what you actually ran.**
- A pure-function assert is not proof that the production path works.
- A passing probe on a copy is not proof that the real code works.
- On every task, state explicitly how you tested and up to which point it is verified.
   - State what you conclude from it.
   - That transparency matters more than hitting any prescribed test shape.

## Verification Levels

**The levels are descriptive and not a mandatory order.**
- Reach the levels the change actually needs, and name which you reached.

| Level | Proves | Does NOT prove |
|---|---|---|
| Pure-function regression guard | The function still returns what it did before, so refactors are protected. | That it meets the real contract. It is a consistency check and not a functional proof. |
| Integration test | The real function, called with real inputs, produced real output that was asserted. | That the user-facing routing, arguments, and auto-detect around it work. |
| Entry-point (CLI / HTTP) | The real caller path works end-to-end, including routing, arguments, and auto-detect. | Visual or rendered correctness that a human must judge. |
| User visual check | The rendered output is right, judged by a human. | Nothing above it. It is the last gate, used only when self-verification is impossible. |

**If you only imported and called the function, say so plainly.**
- The honest phrasing is "verified at integration level, NOT at the entry-point".
- The honest boundary is the deliverable.
   - A green checkmark that overstates it is not.

## Report Honestly

**State the verified boundary before being asked.**
- Spell out exactly how you tested and up to which level it holds.
- Flag any part that was only code-reviewed and not run, up front.

**Separate regression guards from integration tests.**
- In the checklist, split pure-function asserts from tests with real calls and real output.
   - A bare "X/X passing" is not allowed.

**PARTIAL is a legitimate status.**
- It applies when an unrelated cause, like a CAPTCHA or missing data, blocks planned verification.
- Report precisely what ran, what did not, and why.
- Never inflate PARTIAL into "verified" with a weaker substitute that does not test the contract.
- Opus picks the PARTIAL up and decides the next step.

Example reporting format:

```
Tests: 6 integration (passing) + 52 regression-guards (passing).
Verified: URL-transform end-to-end via CLI → real output on disk; blacklist + error path.
NOT verified: visual rendering of the report — needs a user check.
```

## Pre-Commit Checks

**Before the final commit, run the mechanical checks the change actually needs.**
- A mandatory all-five sweep is not required.

1. Syntactic validity, via `python -c "import ast; ast.parse(open('path').read())"`.
2. Imports resolve, meaning every imported module and function exists in the codebase.
3. Library methods exist. For external classes, verify the methods you call with a `dir()` probe. Never trust training data for method names.
4. Pattern compliance, meaning the file structure matches the reference file in sections and style.
5. Edge cases. Verify your parsing handles the data formats the prompt names, like URNs, URLs, and timestamps.
