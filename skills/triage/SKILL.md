---
name: triage
description: >
  A CI job went red and the log is 2,000 lines. Find the ONE real failure buried in the noise,
  rank the likely causes, and give the single first command to run. Use when the user pastes a
  CI/build log, says "why did CI fail", "my pipeline is red", "the build broke", "triage this
  log", "what actually failed", or invokes /triage. Reads the log to the root cause; does NOT
  auto-fix (that's /unfuck) and is for a whole log, not one clean error (that's /tldr-error).
---

# Triage

A CI run failed. The log is a wall of text — setup, install, warnings, a hundred passing lines,
ANSI color codes, and somewhere in there the thing that actually broke. The red X tells you
*that* it failed. It doesn't tell you *why*, and scrolling for the "why" is the tax you pay on
every red build.

This skill does the scrolling. It finds the **real** failure (not the noise it triggered),
names the most likely cause, and hands you **one command to run first.**

**The core move: the first real error is almost always the cause; everything after it is
fallout. Find line one, not line one hundred.**

## Output format

Keep it tight. A red build is an interruption — respect that.

```
**Failed at**: <job / step name> — <the exact log line, quoted>
**Root cause**: <one sentence. what actually broke, not what complained>
**Most likely**: <ranked 1-3 causes, most likely first, each one line>
**Run first**: <the single command or edit to confirm/fix — the highest-leverage next step>
```

If it's a flaky/infra failure, say so plainly instead of inventing a code cause.

## Steps

1. **Find the failing step, not the summary.** CI runners print a failure summary at the *end*,
   but the real error is *above* it — where the failing command first spoke. Scan for the
   earliest of: a non-zero `exit code`, `Error:`, `FAILED`, `AssertionError`, a compiler error,
   `npm ERR!`, `✕`/`✗`, `Process completed with exit code`. The first one in time is your anchor.

2. **Separate the cause from the cascade.** One failure spawns ten downstream complaints —
   "build failed", "job cancelled", "0 tests ran", "step X skipped". Those are *symptoms*. Walk
   **up** from the summary to the first thing that failed on its own. Quote that line, not the
   cascade.

3. **Classify the failure.** Which bucket is it? This decides the fix:
   - **Test failure** — an assertion/expectation broke. Real bug or a stale snapshot.
   - **Build/compile** — type error, syntax, missing symbol. Deterministic; reproduces locally.
   - **Dependency/install** — lockfile drift, yanked version, registry 404, peer-dep conflict.
   - **Environment** — wrong Node/Python/tool version, missing env var or secret, wrong OS.
   - **Timeout/OOM** — job hit a wall/memory limit. Often the *machine*, not the code.
   - **Flaky/infra** — network blip, runner died, race in the test. Passes on re-run.

4. **Rank the likely causes.** Give 1-3, most likely first — grounded in what the log actually
   says, not what usually goes wrong. If the log names a version mismatch, lead with that; don't
   bury it under a generic guess.

5. **Give the single first command.** Not a checklist — the *one* highest-leverage move:
   reproduce it (`npm test -- <file>`), pin it (`npm ci` vs `npm install`), or read it
   (`git log -1 <file>` for what changed). Pick the command that most cheaply confirms cause #1.

## Rules

- **Quote the log, never paraphrase the error.** The exact line is evidence; a reworded version
  is a guess. Cite what's actually there.
- **First error wins.** When several errors appear, the *earliest* is the cause and the rest are
  usually fallout. Don't diagnose line 400 when line 40 already broke.
- **Root cause, not exit status.** "Process completed with exit code 1" is not a diagnosis — it's
  the runner reporting that *something* failed. Find the something.
- **One first command, not a plan.** The point is speed. Rank the causes, then commit to the
  single best next step. `/unfuck` is for when they want it fixed end-to-end.
- **Call flaky *flaky*.** If nothing in the log points to the code — a network reset, a runner
  timeout, a passing local build — say "likely flaky, re-run to confirm" instead of inventing a
  code cause. A false code diagnosis wastes more time than an honest "re-run."
- **Don't invent config you can't see.** If the fix depends on a workflow file or a secret you
  weren't shown, ask for that ONE file — don't guess the YAML.

## Gotchas

- **The summary lies about location.** GitHub Actions' "Annotations" and the final red block point
  at the *step*, not the *line* that failed. The actual error is earlier in the raw log. Always
  scroll up from the summary to the first failure.
- **ANSI color codes.** Raw CI logs are full of `\x1b[31m`-style escape sequences. Read through
  them — the error text is in there; the codes are just coloring. Don't let them hide a keyword.
- **`npm ERR!` is loud, not deep.** npm/yarn print a dozen `ERR!` lines for one failure. The real
  reason is usually the *first* `ERR!` (a version, a 404, a peer conflict); the rest is npm
  narrating its own stack. Same for pip's `ERROR:` spam.
- **`exit code 1` from a test runner ≠ a crash.** It usually means *tests ran and some failed* —
  that's a normal red, look for the failing assertion. A different non-zero code (137 = OOM/killed,
  143 = SIGTERM/timeout) points at the machine, not the tests.
- **"Works on my machine" is a version tell.** If it builds locally but fails in CI, suspect the
  environment first: Node/Python version, a lockfile you didn't commit, `npm install` (drifts) vs
  `npm ci` (pins), or a missing secret. Diff the versions before you touch the code.
- **A cancelled parallel job is a symptom.** In a matrix build, one leg failing can cancel the
  others (`fail-fast`). The cancelled legs aren't the bug — find the leg that failed *first*.
- **Green-then-red with no code change = flaky or upstream.** If the same commit passed before,
  the cause is usually outside your diff: a moved dependency, a registry hiccup, or a real race.
  Re-run once before spelunking.
