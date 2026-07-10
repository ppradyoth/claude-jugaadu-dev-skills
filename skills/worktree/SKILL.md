---
name: worktree
description: >
  Work on a second branch without stashing, committing half-done code, or cloning the repo
  again. Use when the user says "I need to fix a bug on main but I'm mid-feature", "review this
  PR without losing my place", "check out that branch without stashing", "run two branches at
  once", "hotfix without touching my work in progress", or invokes /worktree. Creates a linked
  `git worktree` — a second working directory on the same repo, its own branch checked out — so
  the current one stays exactly as it is. Cleans up safely when done.
---

# Worktree

You're deep in a feature. Files half-edited, nothing ready to commit. Then a bug lands on `main`
that needs a fix *now*, or someone asks you to review a PR. The usual move is `git stash` (and
hope you remember to pop it), or a throwaway commit you'll have to un-commit, or a second `git
clone` that duplicates the whole repo and drifts out of sync.

There's a better tool built into git that almost nobody reaches for: **a worktree is a second
working directory backed by the *same* repository.** Same history, same remotes, same object
store — but a different branch checked out, in its own folder, with its own uncommitted state.
Your feature work sits untouched in the original directory while you fix the bug in the new one.

The whole game: **`git worktree add` a fresh directory for the other branch, do the work there,
then `git worktree remove` it — never a stash, never a second clone.**

## Steps

1. **Confirm it's actually a worktree situation.** The right trigger is: *the user needs a
   different branch checked out but can't disturb the current working directory.* If they've got
   a clean tree and just want to switch, a plain `git switch` is simpler — say so. Worktrees earn
   their keep when there's uncommitted work to protect or two branches genuinely need to be live
   at once (e.g. run the old version and the new side by side).

2. **Pick a location outside the repo.** Put the worktree as a *sibling* of the repo, not inside
   it — a worktree nested in the working tree gets caught by `git add .`, tooling, and watchers.
   The convention is a sibling dir named for the branch:
   ```bash
   # from inside the repo `myapp/`, this creates ../myapp-hotfix
   git worktree add ../myapp-hotfix
   ```
   With no branch argument, git creates a branch named after the directory. Be explicit instead
   (next step) so the branch name is intentional.

3. **Create the worktree with the branch you mean.**
   - **New branch off a base** (the hotfix case) — create and check it out in one shot with `-b`:
     ```bash
     git worktree add -b hotfix/login-500 ../myapp-hotfix origin/main
     ```
     This branches `hotfix/login-500` from `origin/main` (not from your messy feature HEAD — pass
     the base explicitly or you'll inherit your in-progress work).
   - **An existing branch** (review a PR, resume a branch) — name it, no `-b`:
     ```bash
     git worktree add ../myapp-review feature/their-pr
     ```
     A branch can only be checked out in **one** worktree at a time. If it's already live
     somewhere, git refuses — that's the safety feature, not a bug (next rule).

4. **Do the work in the new directory.** `cd ../myapp-hotfix`, fix the bug, commit, push, open the
   PR — all of it happens there. The original directory hasn't moved. Its uncommitted files, its
   branch, its HEAD are exactly where you left them. You can `cd` back and forth freely; each
   directory keeps its own index and its own working state.

5. **List what's live when you lose track.**
   ```bash
   git worktree list
   ```
   Shows every worktree, its path, its HEAD, and its branch. Run it before removing anything so
   you remove the right one.

6. **Remove the worktree when the detour is done.** Don't `rm -rf` the directory — that leaves git
   with a dangling administrative record. Use the porcelain:
   ```bash
   git worktree remove ../myapp-hotfix
   ```
   It refuses if the worktree has uncommitted changes or unpushed commits — deliberately, so you
   don't delete work. Resolve those first (commit/push, or confirm they're throwaway and pass
   `--force`). If a directory was already deleted by hand, `git worktree prune` clears the stale
   record.

7. **Delete the branch if it was a throwaway.** Removing the worktree does **not** delete its
   branch. If the hotfix is merged and the branch is done, clean it up (`git branch -d
   hotfix/login-500`) — or hand off to `/branch-cleanup`, which does exactly this for the whole
   repo.

## Rules

- **Never `rm -rf` a worktree directory.** It orphans the git metadata and leaves `git worktree
  list` lying. Always `git worktree remove` (or `prune` after the fact). The whole reason to use
  worktrees instead of a second clone is that git *manages* them — bypass the manager and you lose
  the benefit.

- **The same branch can't be checked out in two worktrees.** This is a guarantee, not a
  limitation: it's why worktrees are safe. If git refuses to add a branch, it's already live
  somewhere — run `git worktree list` to find where, and either work there or pick a different
  branch. Don't reach for `--force` to override it unless you genuinely understand you'll have two
  working copies of one branch fighting over its state.

- **Branch a hotfix from the *base*, not your current HEAD.** `git worktree add -b fix ../dir`
  with no start-point branches from wherever you are — which, in the mid-feature case, is your
  unfinished work. Pass the base explicitly (`origin/main`) so the fix is clean.

- **Put worktrees beside the repo, never inside it.** A worktree under the main working tree gets
  swept up by globs, build tools, file watchers, and `git add .` in the parent. Sibling
  directories keep the two working copies cleanly separated.

- **Removing a worktree doesn't remove its branch, and vice versa.** They're independent. After a
  merged hotfix you want to remove *both* — the directory (`git worktree remove`) and the branch
  (`git branch -d`). Say which you've done so the user isn't left with a stray branch or a stray
  folder.

- **One repo, local only.** Worktrees all share the one repository's objects and refs — a commit
  in one is visible to all, and they push to the same remotes. This never clones, never touches
  another repo, and never rewrites history. It's a safe, reversible detour: add, work, remove.
