---
name: oops
description: >
  Recover from a botched git operation — a bad reset, a wrong-branch commit, a deleted branch, a
  rebase gone sideways, an accidental `--amend`, a blown-away stash. Use when the user says "oops",
  "undo that", "I messed up git", "I lost my commits", "get my work back", "revert the rebase", or
  invokes /oops. Reads `git reflog` first to find the exact pre-mistake state; never guesses, never
  force-pushes without explicit confirmation.
---

# Oops

Someone just ran a git command and their stomach dropped. A `reset --hard` ate uncommitted-feeling
work. A rebase rewrote ten commits into mush. They committed to `main` instead of their branch. They
deleted a branch that wasn't merged. The work *feels* gone.

It almost never is. Git keeps a log of where every ref has been — the reflog — and nearly everything
short of a `git gc` after the fact is recoverable from it. This skill finds the exact state from
just before the mistake and walks back to it, safely.

The whole game is: **figure out what they actually did, find the last-good SHA in the reflog, and
choose the smallest operation that restores it.** Never destroy more to fix less.

## Steps

1. **Get the story straight.** Ask one question if it's ambiguous: *what was the last command, and
   what did you expect vs. what happened?* Don't act on a vague "it's broken." The recovery for "bad
   reset" is different from the recovery for "wrong-branch commit."

2. **Read the reflog before touching anything.**
   ```bash
   git reflog --date=relative -30        # HEAD's movement history
   git reflog show <branch> --date=relative   # a specific branch's history
   ```
   The reflog is the source of truth. Every entry is a SHA you can return to. The mistake is almost
   always "HEAD moved from GOOD to BAD" — find GOOD.

3. **Confirm the target before moving.** Show the user what the good state contains so they agree
   it's the right one:
   ```bash
   git show --stat <good-sha>     # what that commit is
   git log --oneline <good-sha> -5
   ```
   State plainly: "Your work is at `<good-sha>` — that's the commit before the reset. I'll restore to
   there." Wait for a yes if anything is destructive.

4. **Pick the smallest fix for the specific mistake:**

   - **Bad `reset --hard` / lost commits** → the commits still exist, HEAD just points elsewhere.
     `git reset --hard <good-sha>` (or `git cherry-pick <sha>` to grab just one).
   - **Committed to the wrong branch** → don't reset blindly. Move the commit:
     `git switch right-branch && git cherry-pick <sha>`, then on the wrong branch
     `git reset --hard HEAD~1`. Verify the SHA landed on the right branch before deleting it anywhere.
   - **Rebase went wrong** → `git reflog` shows a `rebase (start)` / `rebase (finish)` boundary; the
     entry just before it is the pre-rebase tip. `git reset --hard <pre-rebase-sha>`. (`ORIG_HEAD`
     often points there too.)
   - **Accidental `--amend`** → the pre-amend commit is dangling in the reflog
     (`HEAD@{1}` right after the amend). Recover it: `git reset --soft HEAD@{1}` to get the original
     message + changes back, or cherry-pick the dangling SHA.
   - **Deleted branch** → the tip SHA is in `git reflog` (search by message) or via
     `git fsck --no-reflogs --lost-found`. Recreate: `git branch <name> <sha>`.
   - **Dropped/cleared stash** → `git fsck --no-reflog --unreachable | grep commit`, inspect with
     `git show <sha>`, restore with `git stash apply <sha>`.

5. **Verify, then stop.** After the fix: `git status` and `git log --oneline -5` to confirm the work
   is back and the tree is clean. Tell the user exactly what state they're in now.

## Rules

- **Reflog first, always.** Never propose a recovery you haven't located in the reflog or via
  `git fsck`. "It's probably at HEAD~1" is a guess; `git reflog` is a fact.
- **Smallest operation wins.** Prefer `cherry-pick` (additive) over `reset --hard` (destructive)
  when you only need one commit back. Don't rewind a whole branch to rescue one commit.
- **Confirm before anything destructive.** `reset --hard`, branch deletion, and force operations get
  an explicit "yes" — show the user the SHA and what it contains first.
- **Never `git gc`, `git prune`, or `reflog expire` during a recovery.** Those are what *actually*
  delete the unreachable objects you're trying to recover. If work is missing, stop the user from
  running them.
- **Force-push only with eyes open.** If the fix rewrites already-pushed history, say so out loud,
  prefer `git push --force-with-lease` over `--force`, and confirm no one else is on the branch.
- **If it's genuinely gone** (committed, never pushed, `gc` already ran, no reflog entry) — say so
  honestly instead of inventing a recovery. Then check editor local-history / IDE backups as a
  last resort.
- Uncommitted changes that were never staged or committed are the one thing the reflog can't bring
  back. Be upfront about that boundary.
