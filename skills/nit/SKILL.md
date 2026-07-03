---
name: nit
description: >
  Catch the embarrassing stuff in your own diff before your reviewer does — leftover
  console.log/print/debugger, a focused test (.only / fit / describe.only), commented-out code,
  a merge marker that slipped through, a TODO you added, an accidentally-committed .env or
  build artifact. Use when the user says "self review", "review my diff", "anything dumb in
  here", "check before I push", "nit", "did I leave anything", or invokes /nit. Mechanical
  self-review only — NOT security (that's /secure-diff) and NOT logic bugs.
---

# Nit

The reviewer hasn't looked yet. Before they do, there's a class of stuff that makes a diff look
sloppy and wastes a review round — a `console.log` you added while debugging, a `describe.only`
that quietly disables the rest of the suite in CI, a block of code you commented out "for now,"
a conflict marker that survived a bad merge. None of it is a *bug*. All of it is a comment
you'll get, a re-push you'll do, and a few minutes of someone's goodwill you'll spend.

This skill reads the diff you're about to push and flags that stuff — **only the lines you
added**, so you're never nagged about code that was already there.

**The core move: review the `+` lines a machine can catch, so the human reviews the logic a
machine can't.**

## Scope

- **In scope:** mechanical smells — debug leftovers, focused/skipped tests, commented-out code,
  merge markers, newly-added TODO/FIXME, placeholder junk, and files that shouldn't be in the
  commit at all (`.env`, build output, huge blobs).
- **Out of scope, on purpose:**
  - **Secrets / injection / unsafe input** → that's **`/secure-diff`**. If you spot a secret,
    say one line ("looks like a key on line N — run `/secure-diff`") and move on; don't
    duplicate that skill.
  - **Logic correctness, does-it-work** → that's a real review or `/quick-test`. `/nit` does
    not reason about whether the code is *right*, only whether it's *tidy*.

## Steps

1. **Get the diff. Added lines only.** Default to what's staged; fall back to the branch.
   ```bash
   git diff --cached                 # staged (default)
   git diff main...HEAD              # whole branch, if nothing staged
   ```
   Work **only** from lines that start with `+` (ignore the `+++` file header). A smell that was
   already in the file is not this PR's problem — flagging it is noise.

2. **Also check what's being added to the tree**, not just line content:
   ```bash
   git diff --cached --name-status   # A = added file, check for junk
   ```
   Flag newly-added files that almost never belong in a commit: `.env`, `.env.*`, `*.log`,
   `dist/`, `build/`, `node_modules/`, `.DS_Store`, `*.pyc`, coverage output, `*.pem`/`*.key`
   (also a `/secure-diff` case), and any single added file over ~500 KB or that reads as
   minified/generated (one enormous line, no newlines).

3. **Scan the added lines for these, in priority order.** Report only what's actually present.

   ### 1. Merge markers (highest — this breaks the build)
   Lines beginning with `<<<<<<<`, `=======`, or `>>>>>>>`. A leftover marker means a conflict
   was half-resolved. This is the one item here that's a hard error, not a nit.

   ### 2. Focused / disabled tests (silently changes what CI runs)
   - **Focused** (runs *only* this test, skips the rest): `describe.only`, `it.only`,
     `test.only`, `fdescribe(`, `fit(`, `fcontext`, Go's `-run` left hardcoded.
   - **Disabled** (skips this test): `xit(`, `xdescribe(`, `it.skip`, `test.skip`, `describe.skip`,
     `@pytest.mark.skip`, `@pytest.mark.xfail`, `t.Skip(`, `@Ignore`.
   Focused especially: a stray `.only` is how a green CI ends up running one test. Always flag.

   ### 3. Debug / print leftovers
   `console.log(`, `console.debug(`, `debugger;`, `print(` (in a language where it's clearly
   debug, e.g. Python app code — not a CLI whose job is printing), `println!(`, `dbg!(`,
   `fmt.Println(`, `p `/`pp ` and `binding.pry` (Ruby), `var_dump(`/`dd(`/`dump(` (PHP/Laravel),
   `System.out.println(`, a bare `puts` in library code, `alert(`.
   Use judgment: a logger call (`logger.info`, `log.Debug`) inside real logging is not a leftover.
   A `console.log` added next to code you were clearly poking at is.

   ### 4. Commented-out code
   Added lines that are **code, commented out** — not prose comments. Tell them apart: a comment
   explaining *why* is fine; a commented-out `// oldFunction(x, y)` or a block of `#`-prefixed
   statements is the "I'll delete it later" that never gets deleted. Flag blocks of 2+
   consecutive commented-out code lines.

   ### 5. Newly-added TODO / FIXME / HACK / XXX
   Only ones **this diff introduces**. Not to forbid them — to make sure you *meant* to ship an
   open TODO, and to suggest turning a real one into a tracked issue.

   ### 6. Placeholder / scaffolding junk
   `lorem ipsum`, `foo`/`bar`/`baz` left in a real (non-test) code path, `asdf`, `test123`,
   hardcoded `http://localhost:PORT` or `127.0.0.1` in shipped code, a hardcoded personal path
   (`/Users/you/...`, `C:\Users\...`), `TODO: remove`, `temporary`, `# HACK`.

4. **Report tight.** This is a pre-push glance, not an audit. If it's clean, say so in one line.

## Output format

```
🔍 /nit — <N> added lines / <M> files reviewed

⛔ Blockers (fix before push)
- <file>:<line> — merge marker: `<<<<<<< HEAD`
- <file>:<line> — focused test: `describe.only(...)` — disables the rest of the suite in CI

⚠️ Nits (probably meant to remove)
- <file>:<line> — debug leftover: `console.log(user)`
- <file>:<line> — commented-out code (4 lines)
- <file>:<line> — added TODO: `// TODO: handle the empty case` → track it?

📦 Files
- Added `.env` — should this be committed? (also a /secure-diff case)

✅ Nothing else stood out. (Logic + security are not this skill's job — /secure-diff for secrets.)
```

If the diff is clean:
```
🔍 /nit — clean. No debug leftovers, focused tests, merge markers, or stray files in the added lines.
```

## Rules

1. **Added lines only.** Never flag a smell that existed before this diff. If it's not on a `+`
   line, it's not yours to nit.
2. **Flag, don't fix.** Report the line and why it bites. Let the user decide — a `console.log`
   might be intentional. (If they say "clean it up," *then* remove them.)
3. **Don't cross into other skills.** Secrets/injection → point at `/secure-diff` in one line.
   Correctness → not your job. Style/formatting a linter owns (spacing, quotes) → skip; a
   formatter handles it, and nagging about it here is noise.
4. **Judgment over grep.** A `print()` in a CLI, a `logger.debug` in real logging, a `TODO` the
   user obviously intends to ship — these are not nits. Match intent, not just the token.
5. **Quiet when clean.** The best result is one green line. Don't invent nits to look useful.
6. **Order by cost.** Merge markers and focused tests first (they break or distort CI), then the
   cosmetic leftovers. A reviewer's time is the thing you're saving.

## Gotchas

- **`.only` is the sneaky one.** It doesn't error — it *passes*, running a single test while the
  suite you think is protecting you sits skipped. A green check with a `.only` in the diff is a
  false green. This is the highest-value catch in the whole skill.
- **Debug prints hide in template strings and JSX.** `{console.log(x)}` in JSX, a `${console.log()}`
  — grep for the call, not just line-start.
- **Not every comment is commented-out code.** A `# returns None on cache miss` is documentation.
  A `# return None  # old behavior` is a leftover. Look for *statements*, not sentences.
- **A committed `.env` is worse than it looks.** Even if you `git rm` it next commit, it's in
  history — treat any secret in it as compromised and rotate. That crosses into `/secure-diff`.
- **Minified/generated files** blow up a diff and are almost never meant to be hand-reviewed. A
  single 40,000-character line is the tell. Flag the file, don't try to read the line.
- **`xit`/`skip` added deliberately** (a known-flaky test quarantined with a linked issue) is
  fine — flag it once so it's visible, don't treat it as a blocker.
