# Testing and Verification

## What a Test Is

**A test depends only on factors you control, with no environment dependency.**
- Repeating it 10 or 100 times in a row produces the same result every time.
- Running it in a month or in a year produces the same result too.

## When a Test Is Required

**A test is required exactly when an implementation changes behavior.**
- The application shows behavior A, and the implementation moves it to A.1 or B.
- The test shows that behavior A.1 or B should match the specification.

**The test of a change also proves that the callers of the changed code behave as before.**
- Every caller is found via import grep and the DOCS.md map.
- The caller check is woven into each change's test individually, so no maintained suite is needed.
- A broken caller is a caused behavior change, so it belongs to the task.

**A test covers the functionality the milestone delivers, and nothing beyond it.**
- An edge case nobody has observed in real data gets no test.
- A test case the implementer invents is not an observation.

## What a Verification Is

**A verification matches the prod environment exactly.**
- It runs once, when tests no longer yield any gain in insight.

**A verification that does not show the intended behavior A.1 or B goes back to the implementation.**
- Return to the implementation change, test again, then verify again.
- A verification that fails twice stops all actions immediately, and you report.

## Evidence Burden

**Complexity that reaches production traces back to a failure observed in real data.**
- A fixture written by the person demanding the defence never counts as that observation.

**A proposed defence carries a measured cost and a measured risk.**
- A measured loss is never traded away for a hypothetical one.

**A demand for proof names a finite, decidable body of evidence.**
- The demand has a stopping condition.
- Asking that a failure be shown impossible is not a valid demand.

## Fallback and Tripwire

**A branch that produces alternative output by a second method is a fallback, and a fallback is eliminated.**

**A branch that refuses to produce output and surfaces the failure is a tripwire, and a tripwire stays.**
- A tripwire never guesses.

**Completeness is a code property, so the safety check lives in a test, never at runtime.**
- The invariant is asserted over a real corpus and kept as a regression.
- Production runs one deterministic route.

**A runtime fallback shipped after a passing proof distrusts the proof.**
- At most a tripwire for genuinely novel input remains.

## dev/

**dev/ holds development scripts for testing, debugging, and experimentation.**

**Permanent value decides what earns a place in dev/.**
- The deciding question is whether the script is useful to another agent with zero context.
- If yes, it belongs in dev/.
- If no, it belongs in the worktree or /tmp/.
- dev/ hands a zero-context agent which tests ran, when, how, and with what result.
