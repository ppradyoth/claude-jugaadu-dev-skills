---
name: branch-cleanup
description: >
  Delete the local branches you're done with — the ones already merged, and the ones whose PR
  merged and remote-deleted (squash/rebase merges show up as `[gone]`). Use when the user says
  "clean up my branches", "delete merged branches", "my branch list is a mess", "which branches
  can I delete", "prune old branches", or invokes /branch-cleanup. Fetch-prunes first, protects
  the current and default branch, deletes safely with `-d`, and never force-deletes without an
  explicit yes.
---

# Branch cleanup

`git branch` scrolls off the screen. Forty local branches, and you know maybe six of them still
matter. The other thirty-four are finished work — PRs that merged weeks ago — but you can't tell
which at a glance, so you leave them all, and the list rots a little more every sprint.

The information to sort them is right there in git. This skill reads it and does the safe deletes,
so the branch list is only branches you're actually on.

The whole game: **fetch-prune to learn what the remote knows, split the branches into merged /
gone / active, and delete only what's provably done — safely, current and default branch always
protected.**

## Steps

1. **Prune first, so `git` knows what the remote deleted.** A merged PR on GitHub usually deletes
   its head branch. Your local copy doesn't hear about that until you prune:
   ```bash
   git fetch --prune                       # drop remote-tracking refs whose upstream is gone
   ```
   Nothing here is destructive to local branches — it only updates `origin/*` bookkeeping.

2. **Find the default branch and the current one — these are off-limits.**
   ```bash
   git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null   # e.g. origin/main → main
   git branch --show-current
   ```
   Never delete either. If `origin/HEAD` isn't set, ask which branch is the trunk rather than
   guessing `main` vs `master`.

3. **Bucket the branches.**
   - **Merged the normal way** (a real merge commit — the branch tip is an ancestor of the trunk):
     ```bash
     git branch --merged <default> | grep -vE '^\*|^\+|(^|\s)<default>$'
     ```
     These are 100% safe: `git branch -d` will delete them and *refuse* if they turn out not to be
     merged, so `-d` is its own safety net.
   - **Gone upstream** (the PR squash- or rebase-merged, so the tips don't match, but GitHub deleted
     the remote branch → it shows as `[gone]`):
     ```bash
     git branch -vv | grep '\[.*: gone\]'
     ```
     This is the signal that catches squash/rebase merges — the case `--merged` can't see, because
     a squash rewrites the commits so the tip is no longer an ancestor of the trunk.

4. **Show the buckets before deleting anything.** List each branch with its last commit date and
   subject so the user recognizes them:
   ```bash
   git for-each-ref --sort=-committerdate refs/heads/ \
     --format='%(refname:short)  %(committerdate:relative)  %(contents:subject)'
   ```
   State plainly: "These 12 are merged into `main` (safe to delete). These 5 are `[gone]` —
   their remote branch was deleted, almost certainly a squash-merged PR. Delete all 17?"

5. **Delete safely.**
   - Merged branches: `git branch -d <name>` (never `-D`). `-d` refuses anything not actually
     merged, so a mislabeled branch survives instead of vanishing.
   - `[gone]` branches: try `git branch -d <name>` first. If git refuses ("not fully merged" —
     common for squash merges, because the squash commit isn't in your local history), **stop and
     confirm** before `git branch -D <name>`. Show `git log --oneline <default>..<name>` so the
     user sees exactly which commits `-D` would drop, and confirm the PR really merged before
     forcing.

6. **Report what's left.** After deleting, `git branch` once more and say what remains and why
   ("6 branches left, all with unmerged local work"). Don't leave the user guessing whether it
   worked.

## Rules

- **`-d`, not `-D`, by default.** The lowercase flag is a safety interlock: it deletes merged
  branches and refuses everything else. Only reach for `-D` on a `[gone]` branch, only after
  showing the commits it would drop, and only with an explicit yes.
- **Never touch the current branch or the default branch.** Filter both out of every list before
  proposing deletions. You can't delete the branch you're on anyway, but don't even list it.
- **`[gone]` means the *remote* is gone, not that the work is safe to lose.** It's a strong hint
  the PR merged (GitHub deletes head branches on merge), but a remote branch can also be deleted
  without merging. Verify with `git log <default>..<branch>` before any force-delete.
- **Prune is not a delete.** `git fetch --prune` only removes `origin/*` tracking refs, never your
  local branches. Say so if the user is nervous — it's the safe first step.
- **Don't delete unmerged work to tidy a list.** A branch with commits not in the trunk and no
  merged PR is live work, however old. Leave it and say why; stale ≠ disposable.
- **Local only.** This skill deletes *local* branches. It never runs `git push origin --delete`
  — removing a remote branch is a separate, louder decision that needs its own confirmation.
