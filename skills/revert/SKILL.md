---
name: revert
description: >
  Undo a commit that's already pushed and other people have — safely, without rewriting shared
  history. Use when the user says "revert that commit", "undo the deploy commit", "back out that
  change", "this broke prod, roll it back", "revert the merge", "undo it but it's already pushed",
  or invokes /revert. Uses `git revert` (a new commit that inverts the old one) — never `reset`
  or force-push on a shared branch. Knows the merge-commit case (`-m`) that trips everyone up.
---

# Revert

A commit is on `main`. It's pushed. Other people have pulled it, CI ran on it, maybe it deployed.
And it's wrong — it broke something, or it shouldn't have gone in.

The instinct is `git reset` to before it, or `--amend`, or a force-push. **On a shared branch,
all three are a mistake.** They rewrite history everyone else already has: the next person to pull
gets a mess of conflicts, and anyone who branched off the commit you deleted is now stranded.
`/oops` uses reset-and-reflog *because that work was local and unshared* — this skill is the
opposite situation.

The right tool is **`git revert`: it doesn't erase the bad commit, it adds a *new* commit that
undoes its changes.** History stays append-only, everyone's clone stays consistent, and the record
honestly shows "we made this change, then we backed it out" — which is exactly what happened.

The whole game: **`git revert <bad-commit>` to move *forward* past a mistake, never `reset` to
pretend it never happened.**

## Steps

1. **Confirm the commit is actually shared.** The dividing line is: *has this commit left your
   machine?* If it's pushed, on a branch others use, or already deployed — revert, always. If it's
   still purely local and unpushed, rewriting is fine and often cleaner; that's `/oops` /
   `/squash` territory, so hand off. When unsure, treat it as shared — revert is safe either way,
   a rewrite of shared history is not.

2. **Identify the exact commit(s).** Find the SHA:
   ```bash
   git log --oneline -15
   ```
   Reverting the most recent commit is the common case. Reverting something buried deeper works
   too, but the further back it is, the more likely its changes conflict with later work — expect
   to resolve conflicts (step 5).

3. **Revert a normal commit.**
   ```bash
   git revert <sha>
   ```
   Git creates a new commit that applies the inverse diff and opens an editor with a prefilled
   message (`Revert "<original subject>"`). Keep it, and add *why* on the next line — a bare
   revert with no reason is a future mystery. To stage the inverse without committing yet (e.g. to
   revert several and commit once), use `git revert --no-commit <sha>` and commit yourself.

4. **Revert a MERGE commit — the one everyone gets wrong.** A merge commit has two parents, so
   git can't guess which side "undo" means. `git revert <merge-sha>` alone **errors**. You must
   name the parent to keep with `-m`:
   ```bash
   git revert -m 1 <merge-sha>
   ```
   `-m 1` means "keep the first parent" — the branch you merged *into* (usually `main`), undoing
   everything the merge brought in. That's almost always what you want. **The catch that bites
   later:** reverting a merge undoes its *changes* but the branch's commits are still in history,
   so if you later re-merge that same branch, git thinks those changes are already present and
   **won't reapply them**. If you'll want the feature back, the fix is to *revert the revert*
   (`git revert <revert-sha>`) when you re-land it — don't just merge again. Say this out loud when
   you revert a merge; it's the #1 surprise.

5. **Resolve conflicts if they come up.** If later commits touched the same lines, the revert
   conflicts like any other. Fix the marked files, `git add` them, then:
   ```bash
   git revert --continue
   ```
   To back out entirely, `git revert --abort` returns to before you started — nothing lost.

6. **Reverting a *range* or several commits.** Give oldest-to-newest and git orders the inverses
   correctly:
   ```bash
   git revert --no-commit <older-sha>..<newer-sha>   # note: exclusive of <older-sha>
   git revert --continue
   ```
   Mind the range: `A..B` excludes `A` itself. To include the whole range down to and including
   `A`, start from `A`'s parent (`A^`). Reverting many at once as one commit is cleaner than a
   pile of individual reverts — use `--no-commit` and commit the batch.

7. **Push the revert like any normal commit.**
   ```bash
   git push
   ```
   No `--force`, because you didn't rewrite anything — you added a commit. This is the whole point:
   a revert is a plain fast-forward push that everyone pulls cleanly.

## Rules

- **Shared history is append-only. Undo it by moving forward, not by erasing.** If the commit is
  pushed or others have it, `git revert` is the *only* correct undo. `reset --hard` + force-push on
  a shared branch rewrites what everyone else already pulled — it turns one person's mistake into
  everyone's merge conflict, and orphans anything branched off the deleted commit.

- **Merge commits need `-m`.** `git revert <merge>` without `-m` fails on purpose — git won't guess
  which parent survives. `-m 1` keeps the branch you merged into (the mainline). Getting this wrong
  either errors loudly (no `-m`) or, with the wrong number, undoes the *opposite* side — check the
  parents with `git show <merge-sha>` if you're unsure which is which.

- **Re-merging a reverted branch silently does nothing.** Because the branch's commits are still in
  history, git sees its changes as already-applied and skips them on a second merge. To bring the
  feature back, *revert the revert* — don't re-merge and wonder why the code didn't come back.

- **A revert is not a delete.** The bad commit and its diff stay in the history and the reflog
  forever — revert removes the *effect*, not the *record*. If the commit leaked a secret, reverting
  does **not** un-leak it: the secret is still in the pushed history and must be **rotated** (and
  the history purged separately). Say so — don't let a revert create false safety.

- **Always say why.** The default `Revert "..."` message states *what*, never *why*. A revert with
  no reason reads, six weeks later, as "someone undid this, no idea if it was on purpose." One line
  — "reverts #812, broke checkout in prod, re-land after fix" — saves the future investigation.

- **Local and unpushed? This is the wrong skill.** If the commit never left your machine, a
  rewrite (`reset`, `--amend`, interactive squash) is cleaner and leaves no revert noise. Reach for
  `/oops` (recover/undo local mistakes via reflog) or `/squash` instead. Revert earns its keep
  precisely when *you can't* rewrite — because the history is already shared.
