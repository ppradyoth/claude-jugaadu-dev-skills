# Prompt Template: Rubber-Duck Debug

For when you're stuck and about to paste a stack trace with "fix this". Don't. This template
makes the model find the *cause* instead of pattern-matching a fix onto the symptom.

## Why this works

Most "fix my bug" prompts get a plausible-looking patch that treats the symptom. You ask why
the null is there; the model adds a null check and moves on. The bug comes back wearing a hat.

Forcing a hypothesis *before* a fix makes the model reason about mechanism. You get a cause,
not a band-aid — and you learn something instead of just unblocking.

## The template

```
Here's a bug. Do NOT write a fix yet.

What's happening:
<observed behavior — the actual wrong thing>

What I expected:
<intended behavior>

Relevant code:
<paste the function + how it's called. Less is more — the smallest reproducer.>

What I've already ruled out:
<anything you tested, so we don't go in circles>

Now:
1. Give me your top 3 hypotheses for the root cause, ranked by likelihood.
2. For each, tell me the ONE thing I could check to confirm or kill it.
3. Wait for me to report what I found before proposing a fix.
```

## Example

> Here's a bug. Do NOT write a fix yet.
>
> What's happening: the retry wrapper fires 4 times on a 401, not the 1 it should.
> What I expected: 401 is non-retryable, so zero retries.
> Relevant code: `<the retry decorator + the call site>`
> What I've ruled out: the 401 itself is correct — the token really is expired.
>
> Now: top 3 hypotheses, the one check for each, then wait.

The model comes back with: maybe the retry predicate treats all 4xx as retryable; check what
status codes the `should_retry` list contains. One look — yep, `range(400, 500)`. Real cause
found in 30 seconds, instead of three rounds of "add a check for 401" whack-a-mole.

## Gotchas

- If you skip "what I've ruled out", expect the model to suggest things you already tried.
- Keep the code paste minimal. A 500-line file buries the signal and burns tokens.
- The "wait before fixing" line matters — without it the model jumps to a patch and you're back to symptom-treating.
