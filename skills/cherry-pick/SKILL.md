---
name: cherry-pick
description: >
  Copy one specific commit (or a few) from another branch onto this one — safely — without merging
  the whole branch. Use when the user says "grab that commit", "cherry-pick that fix", "I need just
  that one commit on this branch", "pull that hotfix into release", "backport it", "apply that commit
  here", or invokes /cherry-pick. Uses `git cherry-pick`. Knows the traps: it makes a NEW commit
  (new SHA — a copy, not a move), the empty/already-applied case, conflicts, and the duplicate-history
  problem when you later merge the source branch.
---

# Cherry-pick

You want *one* commit. Not the branch it lives on, not the five other commits around it — just that
one fix, on the branch you're standing on right now. A hotfix that needs to ride into `release`. A
one-line fix a teammate made on their feature branch that you need on `main` today.

`git cherry-pick` does exactly this: it takes the *diff* of a commit from anywhere in the repo and
replays it as a **new commit on your current branch**.

The one thing to burn into your head first, because every gotcha below flows from it: **cherry-pick
copies, it doesn't move.** The new commit has a **different SHA** from the original. The original
stays exactly where it was. You now have the *same change in two places*, under two different commit
IDs — and git only knows they're related if you tell it (`-x`, step 5).

The whole game: **`git cherry-pick <sha>` to replant one commit's changes here, knowing you've made a
copy — then decide honestly whether that copy will collide with a future merge.**

## Steps

1. **Find the exact commit(s) you want.** Get the SHA from wherever it lives:
   ```bash
   git log --oneline -15 <source-branch>      # e.g. git log --oneline -15 feature/login
   ```
   You don't need to check that branch out. Cherry-pick reaches across the whole repo by SHA.

2. **Be on the branch you want the commit to land on.** This is the #1 mistake — cherry-picking
   onto the wrong branch. Confirm first:
   ```bash
   git branch --show-current      # is THIS where the commit should go?
   ```
   Cherry-pick applies to *wherever HEAD is right now*. If it's not the target branch, `git switch`
   there before you do anything else.

3. **Cherry-pick a single commit.**
   ```bash
   git cherry-pick <sha>
   ```
   Git applies that commit's diff and creates a new commit on your branch, reusing the original
   author, message, and authorship date (the *commit* date is now). Clean apply → done.

4. **Cherry-pick several.** A list, or a range:
   ```bash
   git cherry-pick <sha1> <sha2> <sha3>          # these specific ones, in this order
   git cherry-pick <older-sha>..<newer-sha>       # a range — NOTE: excludes <older-sha> itself
   git cherry-pick <older-sha>^..<newer-sha>      # inclusive of <older-sha> (start from its parent)
   ```
   The `A..B` range is **exclusive of `A`** — the same off-by-one that bites with `git log` and
   `git revert`. To include the oldest commit too, start from its parent with `^`.

5. **Record where it came from — use `-x`.** For anything shared (backport, hotfix into release),
   add `-x`:
   ```bash
   git cherry-pick -x <sha>
   ```
   `-x` appends a `(cherry picked from commit <original-sha>)` line to the message. Since the copy
   has a new SHA, this line is the *only* durable link back to the original — it's what lets a
   future reader (or `git`) know these two commits are the same change. Cheap insurance; use it by
   default on anything that isn't a throwaway.

6. **Resolve conflicts if they come up.** The commit's diff may not apply cleanly on a branch that
   has moved on. Same drill as any conflict:
   ```bash
   # edit the marked files, then:
   git add <files>
   git cherry-pick --continue
   ```
   To bail out and return to before you started, `git cherry-pick --abort` — nothing lost. To skip
   just one commit in a multi-pick and keep going, `git cherry-pick --skip`.

7. **The "nothing to commit" / empty case.** If the change is *already present* on your branch
   (someone cherry-picked it before you, or the branches share it), git stops and says the commit is
   empty. That's git protecting you from a no-op duplicate — don't force it in. `git cherry-pick
   --abort` and move on; the change is already here.

8. **Push like any normal commit.**
   ```bash
   git push
   ```
   You added a commit, you didn't rewrite anything — a plain fast-forward, no `--force`.

## Rules

- **Cherry-pick is a copy, not a move.** The new commit has a **new SHA**; the original stays put.
  You now have the same change twice. This is fine — it's the whole point — but you have to *know*
  it, because the duplicate is what causes the surprise in the next rule.

- **Cherry-picking then later merging the source branch = duplicate commits.** If you cherry-pick a
  commit from `feature` onto `main` today, then next week merge `feature` into `main`, that change
  can land **twice** — once from your cherry-pick, once from the merge — often with a conflict
  because the second copy no longer applies cleanly. Cherry-pick is for when you need a change *now
  and the branches will be merged later*; if the whole branch is coming anyway, prefer to just wait
  and merge. When you must cherry-pick ahead of a merge, `-x` at least leaves a breadcrumb, and
  expect to resolve the redundant hunk at merge time.

- **`A..B` excludes `A`.** The range form skips the oldest commit. For an inclusive range, start
  from the parent: `A^..B`. Getting this wrong silently drops the one commit you probably cared
  about most.

- **Don't cherry-pick a merge commit without `-m`.** A merge has two parents, so git can't guess
  which side's diff you mean; `git cherry-pick <merge-sha>` errors. Name the mainline parent:
  `git cherry-pick -m 1 <merge-sha>`. Usually, though, cherry-picking a merge is a sign you actually
  want the underlying commits — pick those individually instead.

- **Check you're on the right branch before you pick.** Cherry-pick lands on current HEAD. The
  classic failure is running it from the source branch (copying a commit onto itself) or from an
  unrelated branch. `git branch --show-current` first.

- **A cherry-pick with conflicts is still a cherry-pick.** Resolve, `git add`, `--continue`. Don't
  `git commit` manually mid-pick unless you know why — `--continue` finishes it with the right
  metadata (and the `-x` line, if you used it).

## Gotchas

- **The message keeps the *original* author and date, but the commit is yours and now.** So `git
  log` shows the original authorship — good for credit — while the commit's *position* in history is
  today. Don't be surprised the timestamps look out of order.
- **`-x` is not added automatically.** If you forget it, the only trace that two commits are the same
  change is that their diffs match. On a shared/backport pick, that missing breadcrumb is a real
  cost later — make `-x` a habit.
- **Empty-after-pick is a feature, not an error.** Git refusing to create an empty commit means the
  change is already there. Reaching for `--allow-empty` to force it in is almost always wrong.
- **This is the opposite of `/revert`.** `/revert` adds a commit that *undoes* a change; cherry-pick
  adds one that *reapplies* a change somewhere else. If the user actually wants to move a commit off
  a wrong branch (not copy it), that's cherry-pick-here-then-`/revert`-or-`reset`-there, or `/oops`
  territory — say so rather than leaving a stray copy behind.
