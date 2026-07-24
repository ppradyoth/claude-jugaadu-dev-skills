---
name: pre-push
description: >
  The one gate to run before you push: mechanical self-review (/nit), a security pass
  (/secure-diff), and the test suite — in that order, stopping at the first hard blocker.
  Use when the user says "ready to push?", "pre-push", "am I good to push", "final check",
  "ship check", "is this ready", or invokes /pre-push. Answers one question: is this diff safe
  to push, yes or no. Runs the three checks; does not commit (that's /lazy-commit).
---

# Pre-Push

You're about to `git push`. Before you do, three different things can bite you, and they bite in
increasing order of embarrassment:

1. **A sloppy diff** — a `console.log`, a `describe.only`, a survived merge marker. Costs a
   review round.
2. **A leaked secret or an injection sink** — costs a rotated key and a bad afternoon.
3. **A red suite** — costs a broken `main` and everyone's morning.

`/pre-push` runs all three as one gate and gives you a single verdict: **push, or don't.**

**The core move: fail fast at the worst problem. A merge marker or a failing test is a hard
stop — no point security-scanning a diff that won't even build.**

This skill *orchestrates* three skills you already have. It doesn't re-implement them — it runs
them in the right order, short-circuits on a blocker, and collapses their output into one
go/no-go. Think of it as the pre-flight checklist, not a new inspection.

## Scope

- **In scope:** run `/nit`, `/secure-diff`, and the test suite over the diff you're about to
  push; aggregate into one verdict; surface the *first* hard blocker and stop.
- **Out of scope, on purpose:**
  - **Committing** → that's **`/lazy-commit`**. `/pre-push` checks; it doesn't write history.
    (Natural chain: `/pre-push` → `/lazy-commit` → `/pr-desc`.)
  - **Fixing what it finds** → `/pre-push` reports. If the user says "fix it," hand each finding
    to its owner: leftovers to `/nit`, a real bug to `/unfuck`, a red test to `/triage` →
    `/unfuck`.
  - **Deep logic review** → this is a gate, not an audit. It runs the tests; it doesn't reason
    about whether the tests are *right*.

## Steps

1. **Confirm there's something to check.**
   ```bash
   git diff --cached --stat          # staged (default)
   git diff main...HEAD --stat       # whole branch, if nothing staged
   ```
   If both are empty, say "nothing to push" and stop. Note which scope you're on (staged vs.
   branch) and use the same scope for every check below, so all three look at the same diff.

2. **Gate 1 — `/nit` (mechanical).** Run the `/nit` checks over the added lines. Split its
   output into two buckets:
   - **Hard blocker:** a merge marker (`<<<<<<<` / `=======` / `>>>>>>>`) or a focused test
     (`.only` / `fit` / `fdescribe`). These break or silently distort CI. **Stop here** and
     report — don't run the security or test gates on a diff that won't merge or whose green is
     fake.
   - **Soft nits:** debug leftovers, commented-out code, added TODOs, stray files. Collect them;
     they don't stop the gate, they ride along in the final report.

3. **Gate 2 — `/secure-diff` (security).** Run the `/secure-diff` pass over the same diff.
   - **Hard blocker:** a matched secret (real key format) or a clear injection sink on
     attacker-reachable input. **Stop here** — a leaked credential is more urgent than a test
     result, and if it's already committed locally, note that history rewrite + rotation is
     needed.
   - **Soft findings:** lower-confidence "worth a look" items ride along in the report.

4. **Gate 3 — tests (does it actually work).** Detect and run the suite for the project; run only
   what's fast and relevant if the full suite is slow.
   ```bash
   # pick the one that fits the repo — detect, don't guess:
   npm test            # package.json "test" script
   pnpm test / yarn test
   pytest -q           # Python
   go test ./...       # Go
   cargo test          # Rust
   make test           # Makefile target
   ```
   - **Hard blocker:** any failing test, or the suite doesn't run (missing dep, import error).
     A red suite is a no-go. If the failures look pre-existing / unrelated to the diff, say so —
     but still report red, because pushing onto a red base hides your own breakage.
   - If there's **no test suite at all**, say that plainly ("no tests found — can't verify
     behavior") rather than reporting a false green. Absence of tests is a caveat, not a pass.

5. **Verdict.** Combine into one line at the top: **✅ push** or **⛔ don't push**. Then the
   detail, worst-first.

## Output format

```
🚦 /pre-push — <scope: staged | branch main...HEAD>

VERDICT: ⛔ Don't push — 1 blocker

⛔ Blockers (must fix before push)
- [tests] 2 failing — auth/token_test.py::test_refresh, ::test_expiry
- [nit]   src/api.js:88 — focused test `describe.only(...)` (would green a partial suite)

⚠️ Ride-alongs (not blocking, but you probably want to)
- [nit]      src/api.js:41 — debug leftover: console.log(user)
- [secure]   config.js:12 — localhost URL hardcoded — intended for prod?

Next: fix the two blockers, then re-run /pre-push. Want me to start on the failing tests? (/triage → /unfuck)
```

Clean case:
```
🚦 /pre-push — branch main...HEAD

VERDICT: ✅ Push — all three gates green
- /nit: clean (no leftovers, focused tests, or merge markers)
- /secure-diff: clean (no secrets or injection sinks in the diff)
- tests: 128 passed, 0 failed (npm test)

Ready. Next: /lazy-commit → /pr-desc.
```

## Rules

1. **One verdict, up top.** The user asked a yes/no question ("ready to push?"). Answer it in
   the first line, then justify. Don't bury the go/no-go under three sections of detail.
2. **Fail fast, worst-first.** Stop at the first *hard* blocker — a merge marker or a leaked key
   makes the later gates moot. Order the final report by cost: broken build / red tests / leaked
   secret first, cosmetic nits last.
3. **Don't re-implement the sub-skills.** `/pre-push` *calls* `/nit` and `/secure-diff`. If their
   logic changes, this skill inherits it for free. Keep the detection rules in one place.
4. **Same diff for every gate.** Decide staged-vs-branch once, at step 1, and hold it. Scanning
   the branch for secrets but only staged lines for nits gives an inconsistent verdict.
5. **Check, don't commit, don't fix.** Report and stop. Offer the fix chain (`/nit`,
   `/unfuck`, `/triage`) but only run it if the user says go. The gate's job is the verdict.
6. **No false green.** No tests, a skipped suite, an unrun security pass — say it's *unverified*,
   never green. A confident ✅ over an untested diff is worse than an honest caveat.

## Gotchas

- **A `.only` beats a green suite.** The single scariest false-positive: tests "pass" because
  `describe.only` ran one of them and skipped the other 200. `/pre-push` treats a focused test as
  a hard blocker *even if the suite is green* — the green is the lie. This is why the nit gate
  runs before you trust the test gate.
- **"Tests pass" on a suite that didn't run isn't a pass.** An import error, a missing dev
  dependency, or a `0 tests ran` is a red, not a green. Read the runner's summary line, don't
  just check the exit code — some runners exit 0 on "no tests collected."
- **Pre-existing red base.** If the suite was already failing before your diff, your push isn't
  what broke it — but you still can't verify *your* change on top of noise. Report red, name the
  failures as likely pre-existing, and let the user decide. Don't silently pass.
- **Slow suites.** If the full suite takes ten minutes, run the fast/affected subset for the gate
  and say explicitly that you ran a subset — a partial test pass is a caveat, not a full green.
- **Secret already in a local commit.** If `/secure-diff` finds a key on a branch you've already
  committed (not just staged), removing it from the working tree isn't enough — it's in history.
  Flag that the commit needs rewriting and the key needs rotating, then treat it as compromised.
- **This is a gate, not a guarantee.** Three green checks mean "no known mechanical, secret, or
  test-level problem" — not "correct." Logic review is still a human's job (or a real review).
  Say "ready to push," not "this is correct."
