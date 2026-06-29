---
name: why
description: >
  Explain why a line, block, or file exists — the history and intent behind code, not what it does.
  Use when the user asks "why is this here?", "why was this added?", "who wrote this and why",
  "is it safe to delete this?", "what was this hack for", or invokes /why. Chains
  `git blame` → the commit → the PR/issue to recover the original reason. Reads real history;
  never invents a rationale. The companion to /explain (what code does) and /bisect (when it broke).
---

# Why

Someone is staring at a line of code and asking the only question the code itself can't answer:
*why is this here?* A weird `+ 1`. A defensive null check that looks pointless. A commented-out
block. A workaround with no comment. Deleting it feels risky precisely because nobody remembers
what it was for.

`/explain` tells you **what** code does by reading the code. `/bisect` tells you **when** behavior
changed. This skill answers **why** the code is the way it is — by reading the *history*, not the
code. The reason is almost never in the file. It's in the commit that added the line, the PR that
merged it, and the issue that PR closed.

The whole game is: **walk from the line back to the intent — blame the line, read the commit that
introduced it, follow that to the PR/issue — and report the real reason or say plainly that the
trail went cold.** Never fabricate a rationale that sounds plausible.

## Steps

1. **Pin down the target.** A specific line range, a function, or a whole file. If the user points
   vaguely ("this hack"), confirm the exact lines first — blame is line-precise and the answer
   changes line to line.

2. **Blame the line(s) to find the introducing commit.**
   ```bash
   git blame -L <start>,<end> -- <file>          # who/what last touched these lines
   git log -L <start>,<end>:<file>               # the full change history of just this range
   ```
   `git log -L` is the power move — it shows every revision of that exact range, newest first, so
   you see not just the last touch but the line's whole life.

3. **Don't stop at a move/reformat commit.** The most recent blame is often a rename, a reformat,
   or a bulk move that didn't change intent. If the introducing commit looks cosmetic, dig past it:
   ```bash
   git log --follow -p -- <file>                 # history across renames
   git blame <commit>~1 -L <start>,<end> -- <file>   # re-blame the parent to skip a cosmetic touch
   ```
   Use `git blame -w -M -C` to ignore whitespace and detect moved/copied code so you land on the
   commit that wrote the logic, not the one that shuffled it.

4. **Read the introducing commit in full.**
   ```bash
   git show <sha>                                # message + full diff + context
   ```
   The commit *message* is the first-class source of intent. Read it before theorizing. A good
   message says why; a bad one ("fix", "wip") means you keep walking to the PR.

5. **Follow the commit to the PR / issue.** This is where the real reasoning usually lives —
   review discussion, the bug it fixed, the customer report.
   ```bash
   git log --merges --ancestry-path <sha>..HEAD -1   # find the merge commit that pulled it in
   ```
   The merge commit message typically names the PR (`Merge pull request #123`). If a GitHub MCP
   tool or `gh` is available, fetch that PR/issue and read the description + linked issues. Quote
   the relevant line back to the user with a link.

6. **Report the reason, with provenance.** State *why* in one or two sentences, then back it with
   the evidence: the commit SHA + date + author, and the PR/issue number if found. Make the chain
   visible so the user can verify it:
   > This null check was added in `a1b2c3d` (2024-03, PR #218) to fix issue #211 — `parseConfig`
   > returned `null` for empty files and downstream code assumed an object. It's still load-bearing;
   > the empty-file path hasn't changed.

7. **If the trail goes cold, say so.** No useful message, no PR, history truncated by a squash or a
   repo import — report exactly how far you got and stop. "Introduced in the initial commit / a
   squashed import with message 'migrate', no PR — original intent is unrecoverable from history"
   is a true, useful answer. A confident made-up reason is the one failure mode that matters here.

## Rules

- **History is the source, not the code.** The answer to "why" lives in commits, PRs, and issues —
  never reverse-engineer a motive from what the code currently does and present it as fact. That's a
  guess dressed as history.
- **Walk past cosmetic commits.** A reformat/rename/move is not "why this exists." Use `-w -M -C`
  and re-blame the parent until you reach the commit that introduced the *logic*.
- **Distinguish what you found from what you inferred.** "The commit says X" is evidence. "This was
  probably for Y" is inference — label it as such, and only offer it when the trail genuinely ran out.
- **Always show provenance.** SHA + date + author at minimum; PR/issue number when found. The user
  should be able to click through and confirm. An unsourced "why" is worthless here.
- **Answer the deletion question directly when asked.** "Is it safe to remove?" deserves a verdict
  grounded in the reason: if the original cause still exists, say keep it; if the reason is obsolete
  (the bug it guarded is gone, the API it worked around changed), say so — and point at how to verify
  before deleting (a test, the now-fixed upstream).
- **One range at a time.** Blame is line-precise; a function can have five lines added for five
  different reasons across five commits. Don't blur them into one story — break it out if it matters.
- **Read-only by default.** This skill investigates; it doesn't edit. If the user then wants the code
  changed or removed, hand off to the appropriate skill (`/unfuck`, a normal edit) rather than acting
  inside the investigation.

## Example

```
User: /why is there a `time.sleep(0.2)` in the upload retry loop? line 88 of uploader.py

Claude:
  git blame -L 88,88 -- uploader.py        → 7f3a9c1, Priya, 2024-09-12
  git show 7f3a9c1                          → "Throttle S3 retries — fixes intermittent 503s (#604)"
  PR #604 / issue #598                      → S3 returned SlowDown (503) under burst; added 200ms
                                              backoff between retries. Confirmed fix in the issue.

  → The sleep is a 200ms backoff added in 7f3a9c1 (PR #604) to stop S3 SlowDown/503 errors under
    burst uploads. It's still load-bearing — the retry loop has no other backoff. If you want to
    remove it, replace it with proper exponential backoff rather than dropping the delay outright,
    or you'll reintroduce #598.
```
