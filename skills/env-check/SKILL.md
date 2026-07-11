---
name: env-check
description: >
  Find the environment variables your app needs but your local `.env` is missing — before the
  app crashes on a `undefined is not a function` three screens into startup. Use when the user
  says "the app won't start after I pulled", "what env vars am I missing", "check my .env",
  "env-check", "compare my env to the example", "why is this config undefined", or invokes
  /env-check. Diffs the committed `.env.example` (the contract) against the real `.env` (what you
  have), and — better — against the vars the code actually reads, then reports what's missing.
  Read-only: never prints secret values, never writes your `.env`.
---

# Env-Check

You pull `main`, run the app, and it dies. Not with "you're missing `STRIPE_SECRET_KEY`" — with a
null-pointer six calls deep, because someone added a new required env var last week and your local
`.env` predates it. The `.env.example` in the repo knows. Your `.env` doesn't. Nothing told you.

This is the most avoidable startup failure there is. The contract for "what config this app needs"
is right there in the repo — `.env.example`, or the set of `process.env.X` / `os.environ["X"]`
reads scattered through the code. The bug is only that nobody compared it to what you actually
have.

**The core move: treat `.env.example` as the required-keys contract, diff your real `.env`
against it, and report the gap — keys the example declares that your env is missing. Then
cross-check against what the code actually reads, because the example drifts.**

## Scope

- **In scope:** find the env contract (`.env.example` / `.env.sample` / `.env.template`, and the
  `env.*` reads in the code); compare it to the developer's real `.env`; report **missing keys**,
  **extra keys**, and **keys the code reads that nothing declares**. Read-only, names only.
- **Out of scope, on purpose:**
  - **Printing or logging secret values.** This skill compares *key names*. It never echoes a
    value — not the example's placeholder, and absolutely not the real `.env`'s secrets. A skill
    that leaks the thing it's protecting is worse than no skill.
  - **Writing or "fixing" the `.env`.** It tells you what's missing. *You* fill in the real
    values — they're secrets only you (or your secret manager) have. Auto-filling with the
    example's placeholders would hand you a `.env` that looks complete and fails at runtime.
  - **Validating that a value is *correct*** (a live API key, a reachable URL). That's a runtime
    concern; this is a pre-flight name check.

## Steps

1. **Find the contract file.** In priority order:
   ```bash
   ls -a | grep -E '^\.env\.(example|sample|template|dist)$'
   ```
   If several exist, prefer `.env.example`. If none exists, skip to step 4 (code-read fallback) —
   the absence of an example file is itself worth reporting.

2. **Find the developer's real env file.** Usually `.env` (and it should be git-ignored — if
   `git check-ignore .env` comes back empty, flag it, that's an `/untrack` situation). Some
   projects layer `.env.local`, `.env.development`; note which one you're checking.

3. **Extract key names from each, values stripped.** Parse only the `KEY=` on non-comment,
   non-blank lines — take the text left of the first `=`, discard everything right of it:
   ```bash
   # key names only — the cut at '=' means no value ever reaches output
   env_keys() { grep -vE '^\s*(#|$)' "$1" | sed -E 's/^\s*(export\s+)?([A-Za-z_][A-Za-z0-9_]*)\s*=.*/\2/' | sort -u; }
   comm -23 <(env_keys .env.example) <(env_keys .env)   # in example, NOT in your .env → MISSING
   comm -13 <(env_keys .env.example) <(env_keys .env)   # in your .env, NOT in example → extra/stale
   ```
   The `comm -23` line is the answer to "what will crash me." Report those first.

4. **Cross-check against what the code actually reads.** The example file drifts — a var gets used
   in code before anyone updates `.env.example`. Grep the real reads and compare:
   ```bash
   # JS/TS: process.env.FOO / process.env['FOO'] ; Python: os.environ['FOO'] / os.getenv('FOO')
   grep -rhoE "process\.env\.[A-Z0-9_]+|process\.env\[['\"][A-Z0-9_]+['\"]\]|os\.(environ\[|getenv\()['\"][A-Z0-9_]+['\"]" src/ \
     | grep -oE '[A-Z][A-Z0-9_]{2,}' | sort -u
   ```
   Adapt the pattern to the stack (`ENV.fetch(...)` in Ruby, `os.Getenv("...")` in Go,
   `System.getenv("...")` in Java, `env::var("...")` in Rust). Keys the **code reads but the
   example doesn't declare** are the sneakiest gap — the example lies by omission, so a fresh
   clone that trusts it still crashes. Report those loudly.

5. **Report three buckets, names only.** Keep it a scannable triage, not prose:
   - **🔴 Missing (in example / read by code, not in your `.env`)** — fill these before you run.
     If you can tell required from optional (the example groups them, or the code has a default /
     `?? fallback`), say which are hard-required vs. safe-to-skip.
   - **🟡 Undeclared (read by code, in no example)** — the contract is stale; worth a PR to add
     them to `.env.example` so the next person isn't ambushed.
   - **⚪ Extra (in your `.env`, nowhere else)** — probably stale local vars; harmless, but flag in
     case one is a since-removed feature flag.

6. **Say what to do, not just what's wrong.** End with the concrete next step: "add these 3 keys
   to your `.env` and re-run," or "your `.env` matches the contract — the startup crash is
   something else, check X." A clean result is a real result: report it plainly.

## Rules

- **Never print a value — from either file.** The entire parse pipeline cuts at the first `=` on
  purpose. If you find yourself about to show a line that still has a value on it, you have a bug.
  Compare and report **key names only**, always. This is the one rule that makes the skill safe to
  run on a file full of production secrets.

- **Never write the `.env`.** Report the missing keys; let the human supply real values from their
  password manager / secret store. Auto-populating with example placeholders produces a file that
  *looks* configured and fails at runtime with a confusing error — strictly worse than an honest
  "missing."

- **`.env.example` is a hint, not the truth — the code is the truth.** The example drifts behind
  what the code reads. Always run the step-4 code grep; the "read by code, declared nowhere" bucket
  is exactly the gap that a diff-against-example-only check misses.

- **Confirm `.env` is git-ignored while you're here.** If it isn't, the developer is one `git add
  .` from committing secrets — surface it and point at `/untrack`. Don't fix it silently; it
  changes what's tracked.

- **Match the stack before grepping.** `process.env` finds nothing in a Python or Go repo. Detect
  the language from the files present and use the right read-pattern, or you'll report "code reads
  no env vars" on a repo that reads dozens.

- **Read-only, local, reversible.** This skill inspects two files and greps the source. It never
  writes, never installs, never sends anything anywhere. Worst case it tells you nothing you didn't
  know; it can't damage anything.
