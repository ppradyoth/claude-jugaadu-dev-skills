---
name: bisect
description: >
  Find the exact commit that introduced a bug using git bisect — driven, not explained. Use when
  the user says "when did this break", "find the commit that broke X", "bisect this", "it worked
  last week, now it doesn't", or invokes /bisect. Automates the bisect loop with a test command
  when one exists; falls back to guided manual steps. Reports the culprit commit, never guesses it.
---

# Bisect

"It worked last week." Now it doesn't. Somewhere in the last N commits, one of them broke it.
`git bisect` finds that commit in `log2(N)` steps instead of reading every diff. This drives the
loop so the user doesn't have to remember the incantation.

The answer is the commit git lands on. Never name a culprit from reading diffs — that's a guess.
Bisect proves it.

## Steps

1. **Pin down "broken" and "working".**
   - **Bad** = a commit where the bug is present. Default to `HEAD`.
   - **Good** = a commit where it worked. Ask the user for one (a tag, a date, "last release"), or
     find a candidate: `git log --oneline --before="1 week ago" -1`.
   - Confirm the good commit is actually an ancestor of bad: `git merge-base --is-ancestor <good> <bad>`.

2. **Get a yes/no test for the bug.** Bisect needs one question answered at each step: *is the bug
   here?* The cleaner this is, the faster and more trustworthy the result.
   - Best: a command that exits `0` when good, non-zero when bad — a failing test, a build, a script.
     Narrow it to the one case (`pytest path::test_x`, `npm test -- -t "name"`), not the whole suite.
   - If the symptom is manual ("the page is blank"), that's fine — you'll answer good/bad by hand
     each step.

3. **Run the bisect.**
   ```bash
   git bisect start
   git bisect bad <bad>            # or just: git bisect bad
   git bisect good <good>
   ```

4. **Drive the loop.**
   - **Automated** (you have a test command): let git do every step.
     ```bash
     git bisect run <your test command>     # e.g. git bisect run pytest -x path::test_x
     ```
     Exit code rule git follows: `0` = good, `1–124/126/127` = bad, `125` = skip (can't test this
     commit). Make the command match that.
   - **Manual**: at each commit git checks out, run/inspect, then tell git the verdict:
     ```bash
     git bisect good     # bug NOT present here
     git bisect bad      # bug IS present here
     git bisect skip     # can't tell (won't build, unrelated breakage)
     ```

5. **Report the culprit.** Git prints `<sha> is the first bad commit`. Show:
   - The commit: `git show <sha> --stat` — author, date, message, files touched.
   - A one-line read of *why* it likely caused the bug, pointing at the specific changed lines.
   - The fix direction (revert, or the targeted change) — but as a suggestion, clearly separate from
     the proven fact of which commit it was.

6. **Always clean up.** `git bisect reset` returns you to where you started. Do this even if the run
   errored — a half-finished bisect leaves the repo on a detached checkout.

## Rules

- The culprit is whatever `git bisect` lands on. Don't pre-announce a guilty commit from skimming
  the log — run the bisect and let it prove it.
- One symptom, one test. A flaky or broad test makes bisect lie. If the test is flaky, say so and
  prefer manual verdicts.
- Use `git bisect skip` (not a guessed good/bad) for commits that won't build or have unrelated
  breakage — a wrong verdict sends bisect down the wrong half and the result is garbage.
- `git bisect run` returns control to the human between automated runs only if it can't decide —
  trust its exit-code contract; don't babysit each step when a command can answer.
- Never leave the repo mid-bisect. `git bisect reset` at the end, always.
- If `good` isn't an ancestor of `bad` (e.g. across a rebase or unrelated branch), bisect can't run
  cleanly — say so and help find a real common-ancestor good commit instead.
