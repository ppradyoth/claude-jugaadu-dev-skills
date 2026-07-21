---
name: amend
description: >
  Fix the *last* commit you just made — add a file you forgot, fold in a one-line fix, or correct
  the commit message — without stacking a "fix typo" or "add missing file" commit on top. Use when
  the user says "amend that", "add this to the last commit", "I forgot a file", "fix the last commit
  message", "reword my last commit", "I committed too early", "wrong author on that commit", or
  invokes /amend. Uses `git commit --amend`. Knows the traps: amend rewrites the SHA (force-with-lease
  if it's pushed), it folds in *staged* changes only, and it touches HEAD alone — an older commit is
  `/rebase` / `/squash`, not this.
---

# Amend

You commit, then half a second later you see it: a file you forgot to stage, a typo in the message, a
stray `console.log` you meant to delete. The clumsy fix is a second commit — `fix: add missing file`,
`typo` — that clutters history forever. `git commit --amend` replaces the last commit in place instead,
so it lands the way you meant it the first time.

The one thing to burn in first, because the worst gotcha flows from it: **`--amend` does not edit the
last commit — it *replaces* it with a brand-new commit that has a new SHA.** The original commit isn't
modified; it's abandoned (it lingers in the reflog, recoverable, but off the branch). So amending a
commit you've already **pushed** makes your branch diverge from the remote — a plain `git push` will be
rejected, and only `--force-with-lease` gets it up cleanly.

The whole game: **stage exactly the fix, `git commit --amend` (with `--no-edit` if the message is
fine), and force-with-lease *only if it was already pushed*.**

## Steps

1. **Confirm it's the LAST commit you're fixing, and see what's currently staged.** Amend only ever
   touches `HEAD`. And it folds in whatever is staged *right now* — so look before you amend:
   ```bash
   git log --oneline -1        # this is the commit --amend will replace
   git diff --staged           # exactly what will get folded into it — nothing more, nothing less
   ```
   If the commit you want to fix is older than HEAD, stop — that's `/squash` or a `rebase` fixup, not
   amend (see Gotchas).

2. **Stage the fix — precisely.** Amend absorbs the staged changes into the last commit. Stage *only*
   what belongs there:
   ```bash
   git add path/to/forgotten-file.js      # the file you forgot
   git add -p src/thing.js                # or just the hunk that fixes the goof
   ```
   Anything you leave unstaged stays a separate change in your working tree — it does **not** go into
   the amended commit. (Amend does not sweep up unstaged edits.)

3. **Amend.** Three shapes, depending on what you're fixing:
   ```bash
   git commit --amend --no-edit           # add the staged fix, KEEP the existing message (most common)
   git commit --amend                      # add the staged fix AND open the editor to edit the message
   git commit --amend -m "feat: correct message"   # fix ONLY the message (nothing staged? just rewords)
   ```
   With nothing staged, `--amend` becomes a pure message reword. With something staged, `--no-edit`
   folds it in silently; drop `--no-edit` when you also want to touch the message.

4. **Verify the result before you push.** The SHA changed — check the commit is what you intended:
   ```bash
   git show --stat HEAD        # new SHA, the forgotten file now present, message right
   ```

5. **Push — and here's the fork that matters:**
   ```bash
   # NOT pushed yet (local-only commit): just push normally
   git push

   # ALREADY pushed: the remote still has the old SHA — you must overwrite it, safely
   git push --force-with-lease
   ```
   `--force-with-lease` refuses the push if someone else added commits you haven't seen (unlike a blind
   `--force`, which would stomp them). If it's rejected, fetch and look before forcing again — see Rules.

## Rules

- **Amend rewrites history — obey the golden rule.** Because `--amend` makes a *new* SHA, only amend a
  commit that is **yours alone and not yet built on by anyone else**. Amend a commit others have already
  pulled and you've split the history: their branch still has the old commit, yours has the replacement,
  and the next merge tangles both. Same rule as `/rebase`. Safe to amend freely while the commit is
  local-only; think twice once it's shared.

- **Never plain `--force` an amended push — use `--force-with-lease`.** After amending a pushed commit,
  the remote branch and yours have diverged, so a normal push is rejected. `--force-with-lease` overwrites
  the remote *only if it still points where you last saw it* — so it won't silently destroy a teammate's
  commit that landed in between. Plain `--force` skips that check and can erase work. If lease is
  rejected, `git fetch` and inspect (`git log origin/<branch>`) before deciding.

- **Amend folds in staged changes only — check `git diff --staged` first.** It's easy to have unrelated
  edits staged from earlier and quietly bundle them into the last commit. What's staged when you run
  `--amend` is exactly what gets absorbed; unstaged changes are left in the tree. One glance at
  `git diff --staged` before amending prevents a polluted commit.

- **Amend touches HEAD and nothing deeper.** It can only fix the *most recent* commit. To fix a commit
  three back, don't amend — that's a `rebase` autosquash (`git commit --fixup=<sha>` then
  `git rebase -i --autosquash`) or `/squash` if you're collapsing the lot. Amending won't reach it.

- **Fixing author/date needs `--reset-author`.** Amend keeps the *original* author and author-date by
  default, even if you fixed your `user.email` since. If the commit went out under the wrong identity,
  `git commit --amend --reset-author --no-edit` stamps it with your current name/email and now. Without
  that flag the wrong author sticks.

- **A wrongly-amended commit isn't gone — the reflog has it.** Amended over the wrong thing, or force-pushed
  a mistake? The pre-amend commit is still in `git reflog` as a dangling commit for a while
  (`HEAD@{1}` right after). Recovering it is `/oops` territory — don't panic and re-commit blindly.

## Gotchas

- **The big one: amend on an already-pushed commit needs `--force-with-lease`.** A normal `git push` after
  amending is rejected as non-fast-forward — that rejection is the tell that you rewrote a pushed commit,
  not a network error. Force-with-lease, never blind force.
- **"Amend" ≠ "fix an old commit."** Amend is *only* the last one. If what you actually want is to edit a
  commit further back, or squash several, that's `/squash` (many of yours → one) or a `rebase --autosquash`.
  Reaching for amend there just rewrites the wrong commit.
- **Unstaged changes don't ride along.** People expect `--amend` to pick up all their working-tree edits;
  it only takes what's **staged**. Forgot to `git add` the fix and it won't be in the amended commit —
  you'll amend an empty change (just a reword). Stage first.
- **`--amend` with nothing staged is a pure reword.** Handy — `git commit --amend -m "..."` (or without
  `-m` to open the editor) fixes just the message. But it *still* makes a new SHA, so the pushed-commit
  force-with-lease rule applies even when all you changed was a typo in the message.
- **Amend after a bad merge/rebase, and you may be papering over `/oops`.** If the last "commit" is really
  a botched rebase or merge state, amending it compounds the mess. When the last commit looks wrong because
  a git operation went sideways, read the reflog first (`/oops`) rather than amending on top.
