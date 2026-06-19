# Verification — Test the Real Path, Not a Parallel Reimplementation

## Four Patterns

### Pattern 1 — Wrong Path: Tests Check a Parallel Reimplementation, Not the Production Function

When you build a function in `src/foo.py` and a dev script duplicates similar logic inline for a probe, the probe is **not a regression check** for the `src/` function. It tests the parallel code path.

**Rule:** when you migrate or change a function in `src/`, at least **one** verification must call the real `src/` function — import it directly, with real inputs, with an assertion on the real output. Not a function with the same name in a probe script. Not a reimplementation that "should be equivalent". The real one.

### Pattern 2 — Trivial Asserts Against Pure Functions Counted as "Tests"

```python
def test_url_transform(self):
    assert apply_transform("https://example.com/abs/12345") == "https://example.com/pdf/12345"
```

This is a code-consistency check ("the function returns what we put in"), not a contract validation against reality — a refactor guard, **not** a functional proof.

**Rule:** when reporting tests in a Completion Checklist, separate **regression guards** (pure-function asserts without contract validation) from **integration tests** (call → real I/O → assert on outcome). Example format:

```
Tests: 6 integration (passing) + 52 regression-guards (passing).
Integration: URL-transform end-to-end workflow → real output on disk;
            blacklist + error path; multi-step chain.
Regression-guards: pure-function asserts on transforms/filter/regex behavior.
```

Never just "X/X passing" without this separation. The user should see at a glance what was actually verified vs what is boilerplate.

### Pattern 3 — PARTIAL Without Fallback Verification

When the planned verification fails due to an **unrelated** cause (CAPTCHA hang, server 503, test data missing), the answer is NOT "report PARTIAL and stop". The answer is: find a smaller or alternative verification that still hits the contract.

**Rule:** "PARTIAL" as a verification status is acceptable only when you have attempted at least 2 alternative smaller verifications and list in the report why each failed. The default is: break the verification into smaller parts that can run individually, rather than giving up.

### Pattern 4 — User-visible Entry Point Skipped

Code that has a CLI/HTTP entry-point MUST be verified at least once via that entry-point. Direct Python import + function call is NOT enough — the CLI wrapper path often has its own routing/argument/auto-detect logic that a direct import bypasses.

**Rule:** the Completion Checklist must include a line that explicitly invokes the user-facing entry-point: a `<cli-tool> ...` call or a curl against the HTTP endpoint. Not just a Python import. The output of that call belongs in the checklist (truncated).

## What Verification "Done Right" Looks Like

Your Completion Checklist should contain, in this order:

1. **Pure-function regression guards** — briefly named with count, clearly labeled as such. Example: "Regression-guards: 52 pure-function asserts (transforms, blacklist, regex behavior). Passing."
2. **Integration tests against the real src/ function** — at least one per new/changed function. With concrete input and concrete outcome assertion.
3. **End-to-end verification via the user-facing entry-point** — CLI call or HTTP request. With real output truncated.
4. **Other verifications** (smoke runs, sample tests) when relevant.

If any step from 1-3 is not possible → explicit entry in the checklist explaining why, plus an alternative that hits the contract instead.

## What This Rule Does NOT Say

- It does not say "don't write unit tests" — write them, but report them for what they are (regression guards), not as "verification".
- It does not say "every test must make network I/O" — pure-function tests have value as refactor protection, they are just not a functional proof.
- It does not say "end-to-end CLI test can never be skipped" — if you have not changed a CLI/HTTP endpoint (e.g. only refactored engine logic), an integration test without a CLI roundtrip is sufficient. But the test must reach the function via the path a real caller would use.
