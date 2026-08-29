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

## What a Verification Is

**A verification matches the prod environment exactly.**
- It runs once, when tests no longer yield any gain in insight.

**A verification that does not show the intended behavior A.1 or B goes back to the implementation.**
- Return to the implementation change, test again, then verify again.
- A verification that fails twice stops all actions immediately, and you report.
