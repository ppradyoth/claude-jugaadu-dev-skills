---
name: rebase
description: >
  Move your branch's commits on top of an updated base (usually `main`) so history stays linear —
  safely. Use when the user says "rebase onto main", "update my branch with main", "my branch is
  behind main", "replay my commits on top of the latest", "rebase my feature branch", "get the
  latest main into my branch without a merge commit", or invokes /rebase. Uses `git rebase` +
  `--force-with-lease`. Knows the golden rule (never rebase shared history), the conflict-per-commit
  loop, `--onto` surgery, and why you never plain `git push --force` after a rebase.
---

# Rebase

Your feature branch is three commits deep. Meanwhile `main` moved on — other people merged things.
Now your branch is *behind*, and you want the new `main` work underneath yours so you can test
against the latest and merge cleanly. Two ways to do that: merge `main` into your branch (leaves a
merge commit and a tangled graph), or **rebase** — lift your commits off, fast-forward your branch to
the new `main`, and replay your commits one by one on top. Linear history, no merge bubble.

That replay is the whole idea, and also the whole danger. **Rebase doesn't move your commits — it
rewrites them.** Each replayed commit is a *new commit with a new SHA*. The originals are abandoned
(recoverable via reflog, but gone from the branch). So a rebased branch has a history that no longer
matches what anyone else may have pulled — which is why the one unbreakable rule below exists.

The whole game: **`git rebase main` to replant your branch's commits on top of the latest base,
knowing you've rewritten them — then push with `--force-with-lease`, and only ever on a branch that's
yours alone.**

## The golden rule (read this before anything else)

**Never rebase a branch other people have based work on or pulled to build on.** Rebase rewrites
SHAs; if someone else has the old commits, their history and yours diverge, and the next time they
pull they get a mess of duplicated commits and conflicts that's genuinely painful to untangle.

The safe zone is a branch that is **yours** — a feature/PR branch only you push to. Rebasing *that*
onto `main` is routine and good. Rebasing `main` itself, or a shared `develop`, or a branch a
teammate has a PR stacked on, is how you ruin someone's afternoon. If you're not sure whether anyone
else has the branch: **merge instead of rebase.** A merge commit is ugly; a rewritten shared branch
is a support ticket.

## Steps

1. **Confirm this branch is yours to rewrite.** The golden rule, made concrete:
   ```bash
   git branch --show-current          # not main / develop / a shared release branch
   ```
   If it's a shared branch, stop — use a merge (`git merge main`) instead and say why.

2. **Start clean.** Rebase refuses to run with uncommitted changes, and a dirty tree makes conflicts
   ambiguous. Commit your work, or stash it:
   ```bash
   git status                          # working tree clean?
   git stash                           # if you have WIP you don't want to commit yet
   ```

3. **Get the latest base.** You can't replay onto commits you don't have locally:
   ```bash
   git fetch origin
   ```

4. **Rebase onto the updated base.** Replay your branch's commits on top of the newest `main`:
   ```bash
   git rebase origin/main
   ```
   Git rewinds your branch to where it diverged, fast-forwards to `origin/main`, then reapplies each
   of your commits in order as a new commit. Clean run → your branch is now `origin/main` + your work
   on top, linear.

5. **Resolve conflicts one commit at a time.** This is the part that surprises people: rebase stops
   at *each* commit that doesn't apply cleanly, not once at the end like a merge. For each stop:
   ```bash
   # edit the conflicted files, then:
   git add <files>
   git rebase --continue               # NOT git commit — --continue finishes this step
   ```
   `git rebase --skip` drops the current commit (rare — only if it's genuinely redundant now).
   `git rebase --abort` returns you to exactly where you started, nothing lost — your escape hatch at
   any point. If the conflicts are gnarly, hand them to **`/conflict`**, which resolves on the merits
   and knows the ours/theirs flip that rebase introduces (see Gotchas).

6. **Push the rewritten branch with `--force-with-lease`.** Your local history no longer matches the
   remote branch (new SHAs), so a normal push is rejected. Force it — but the *safe* force:
   ```bash
   git push --force-with-lease
   ```
   **Never plain `git push --force`.** `--force-with-lease` refuses the push if the remote branch has
   commits you haven't seen (someone else pushed while you were rebasing) — it protects their work.
   Plain `--force` overwrites blindly and can delete a teammate's commit without a trace.

## Rebase vs. merge — which to reach for

- **Rebase** when the branch is **yours** and you want a clean, linear history to merge — "get my PR
  branch up to date with `main` before I merge it." This is the common, good case.
- **Merge** (`git merge main` into your branch) when the branch is **shared**, or you simply don't
  want to rewrite history, or a rebase would be a huge conflict slog across many commits. A merge
  commit is honest about what happened and never rewrites anyone's SHAs.
- Don't rebase to "clean up" a branch other people are already building on. Correctness beats a tidy
  graph.

## Rules

- **Golden rule: only rebase branches that are yours alone.** Rewriting SHAs on anything shared
  diverges everyone else's history. When in doubt, merge.
- **`--force-with-lease`, never `--force`.** After a rebase you *must* force-push your own branch —
  but the leased form aborts if the remote moved under you, so you never clobber a push you didn't
  know about. Plain `--force` is how teammates' commits vanish.
- **`--continue`, not `commit`, mid-rebase.** After resolving a conflict, `git add` then
  `git rebase --continue`. Running `git commit` yourself mid-rebase creates a stray commit and
  confuses the replay. `--abort` is always safe if you want out.
- **Conflicts come per commit, not once.** A five-commit branch can stop five times. That's normal —
  each stop is one of *your* commits being reapplied against the new base. Resolve, continue, repeat.
- **Rebase rewrites; it doesn't move.** New SHAs, old commits abandoned. That's exactly why it's
  unsafe on shared branches and why you can always recover via `git reflog` if a rebase goes wrong
  (that's `/oops` territory).

## Gotchas

- **`ours` and `theirs` are backwards during a rebase.** Because rebase replays *your* commits on top
  of the base, git treats the base as `ours` (it's checked out first) and *your* commit as `theirs`.
  So `git checkout --theirs` keeps *your* change and `--ours` keeps the base's — the opposite of what
  those words mean during a normal merge. Get this backwards and you'll silently discard the wrong
  side. When unsure, resolve by reading the actual code, not by picking a side blindly (`/conflict`
  handles this correctly).
- **The same conflict, over and over, across commits.** If two of your commits both touch a line that
  `main` also changed, you can resolve "the same" conflict at commit 2 and hit it again at commit 4.
  Turn on `git rerere` (`git config --global rerere.enabled true`) once and git remembers your
  resolutions and replays them automatically next time — a big quality-of-life win for anyone who
  rebases regularly.
- **Rebasing a merge commit flattens it.** A plain `git rebase` drops merge commits and replays their
  contents linearly, which is usually fine but occasionally not what you want. `git rebase
  --rebase-merges` preserves the merge structure if you actually need it — rare, but know it exists.
- **`--onto` is for moving a branch to a *different* base entirely.** `git rebase --onto <newbase>
  <oldbase>` transplants only the commits after `<oldbase>` onto `<newbase>` — the tool for "I branched
  off the wrong branch" or splitting a stacked branch off its parent. Powerful and easy to get wrong;
  reach for it deliberately, and back up the branch first (`git branch backup/<name>`).
- **Force-pushing a rebased branch invalidates open review comments.** Because the SHAs changed,
  line-anchored PR comments can go stale or detach. Not a reason to avoid rebasing a PR branch — just
  expect it, and prefer rebasing *before* review starts or *after* it's approved, not mid-review.
- **This is the opposite failure mode from `/squash`.** `/squash` collapses your *own* messy commits
  into one via a soft reset (no replay, no conflicts). Rebase *replays* your commits onto a new base
  (conflicts possible). If the goal is "tidy my commits," that's `/squash`; if it's "get the latest
  base under my work," that's rebase.
