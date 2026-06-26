# Recipe: Hunt a Regression

Something that worked last week is broken now. This is the fastest path from "it's broken" to a
merged fix — find the exact commit that did it, understand it, fix it, ship it. No diff-reading
marathon.

## When to use

A bug that *appeared* — it used to work and now doesn't. (For a bug that was always there, skip
straight to `/unfuck`; there's no regression to bisect.)

## The chain

With `/jugaadu` you can say it in one line:
> "find when the export broke, explain that commit, fix it, and write me a PR"

Or run them in order:

1. **`/bisect`** — find the exact commit that introduced the bug. Give it a known-good point ("it
   worked in last Friday's build") and a yes/no test if you have one. This is the step that turns
   "somewhere in 40 commits" into "this one line."

2. **`/explain`** — point it at the culprit commit. Understand *why* that change broke things before
   you touch it. A fix you don't understand is the next regression.

3. **`/unfuck`** — fix the root cause the bisect found, not the symptom. If a clean revert is
   correct and safe, that's often the answer; if not, target the actual line.

4. **`/quick-test`** — add the test that would have caught this. A regression that ships without a
   test for it is a regression that comes back.

5. **`/pr-desc`** — generate the PR. Reference the culprit SHA so the fix is traceable to the cause.

## Manual fallback (no skills)

```bash
# 1. Find the breaking commit (swap in your own good ref + test command)
git bisect start
git bisect bad
git bisect good <last-known-good-ref>
git bisect run <test command>     # e.g. pytest -x path::test_x
git bisect reset                  # ALWAYS clean up

# 2. Look at what it changed
git show <first-bad-sha> --stat

# 3. After fixing, confirm the test now passes, then push
```

## Gotchas

- Bisect is only as good as the good/bad verdict at each step. A flaky test will point at the wrong
  commit — stabilize the test or bisect manually.
- The "first bad commit" can be a merge commit. If so, the real change is on the branch it merged;
  bisect into that history or read the merge's diff.
- Don't skip step 4. The whole point of finding the exact commit is to write the one test that pins
  the behavior so it can't silently break again.
- `git bisect reset` even if something errored mid-run — otherwise you're left on a detached
  checkout wondering why your branch looks wrong.
