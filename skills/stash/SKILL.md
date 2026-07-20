---
name: stash
description: >
  Shelve your work-in-progress so the working tree goes clean, then bring it back later — without a
  throwaway commit. Use when the user says "stash this", "save my changes for a sec", "I need to
  switch branches but I'm mid-change", "park this and come back", "pop my stash", "get my stashed
  work back", "I stashed something and lost it", or invokes /stash. Uses `git stash`. Knows the traps:
  untracked files aren't stashed by default (`-u`), `pop` on a conflict does NOT drop the stash,
  everything comes back unstaged unless `--index`, and `git stash branch` for when the base has moved.
---

# Stash

You're mid-change and something urgent cuts in — a bug to reproduce on `main`, a PR to check out, a
quick pull. Your working tree is dirty, so you can't switch cleanly, and you don't want a `wip` commit
polluting history. `git stash` shelves the dirty state onto a stack, resets your tree to a clean HEAD,
and hands the changes back whenever you ask.

The one thing to burn in first, because two of the worst gotchas flow from it: **a plain `git stash`
saves your tracked changes and nothing else.** Untracked files (brand-new files git isn't watching yet)
and ignored files are left sitting in the tree. Switch branches thinking you're clean and those new
files follow you across — or get clobbered. Stash the *whole* dirty state and you need `-u`.

The whole game: **`git stash push` to shelve, `git stash pop` to bring it back — knowing exactly what
got shelved (tracked only, unless `-u`) and that `pop` after a conflict leaves the stash in place.**

## Steps

1. **See what you'd be shelving first.** Stash is quiet about what it takes. Look before you leap:
   ```bash
   git status --short        # M = tracked-modified (stashed), ?? = untracked (NOT stashed by default)
   ```
   If there are `??` lines you care about, you need `-u` in the next step. This one check prevents the
   most common stash surprise.

2. **Stash it — with a message, and `-u` if there's untracked work.**
   ```bash
   git stash push -m "wip: half-done login form"     # tracked changes only
   git stash push -u -m "wip: login form + new files" # -u ALSO includes untracked files
   ```
   `push -m` names the stash so `git stash list` is readable later (an unnamed stash shows only the
   branch + commit it was made on — useless once you have three). Use `-u`/`--include-untracked`
   whenever step 1 showed `??` lines. `-a`/`--all` also grabs *ignored* files — rarely what you want,
   don't reach for it reflexively.

3. **Stash only some files** when you want to keep the rest in the tree:
   ```bash
   git stash push -m "just the css experiment" -- src/styles.css src/theme.css
   ```
   Everything not named stays exactly as it is. Good for parking one risky edit while you keep working
   on the rest.

4. **Do the other thing.** Your tree is now clean at HEAD — switch branches, pull, check out the PR,
   reproduce the bug. The stash sits untouched on the stack.

5. **List and preview before you restore.** The stack is newest-first:
   ```bash
   git stash list                    # stash@{0} is the NEWEST, stash@{1} older, ...
   git stash show -p stash@{0}       # full diff of a stash before you apply it
   ```
   `stash@{0}` is always the most recent. The refs *shift* every time you push or drop one — don't
   memorize a number, re-list.

6. **Bring it back — `pop` (apply then remove) or `apply` (apply, keep).**
   ```bash
   git stash pop            # apply stash@{0} and drop it from the stack — the usual move
   git stash apply          # apply but LEAVE it on the stack (restore onto two branches, or hedge)
   git stash pop stash@{1}  # a specific one, not just the newest
   ```
   Prefer `apply` when you're not 100% sure it'll land clean and you want a second try; `pop` once
   you're confident.

7. **Clean up applied/dead stashes.**
   ```bash
   git stash drop stash@{0}   # remove one entry (what pop does automatically on success)
   git stash clear            # wipe the ENTIRE stack — every stash, gone
   ```
   `git stash clear` is the dangerous one — see Rules. Drop individually unless you truly want none.

## Rules

- **A plain stash skips untracked and ignored files.** `git stash` / `git stash push` shelves only
  *tracked* modifications (staged + unstaged). A new file git hasn't started tracking is left in the
  working tree. Add `-u` to include untracked, `-a` to also include ignored. If step 1 showed `??`
  lines and you stashed without `-u`, those files did **not** go into the stash — check before you
  assume the tree is clean.

- **`pop` on a conflict does NOT drop the stash.** If applying a stash hits a merge conflict, git
  applies what it can, leaves conflict markers — and **keeps the stash on the stack** (it only auto-drops
  on a *clean* apply). People see the conflict, fix it, and think the stash is gone; it isn't. After you
  resolve (`git add` the files), the stash is still `stash@{0}` — remove it yourself with
  `git stash drop`. Don't blindly `pop` again.

- **Everything comes back unstaged.** By default `apply`/`pop` restore all your changes as *unstaged*,
  even the parts that were staged when you stashed. If you need the staged/unstaged split preserved,
  use `git stash apply --index` (or `pop --index`). Without it, re-staging is on you.

- **When the base has moved, use `git stash branch` instead of fighting conflicts.** If HEAD advanced a
  lot since you stashed, `pop` can conflict badly. `git stash branch <newname> [stash@{n}]` creates a
  branch **from the commit the stash was made on**, checks it out, and applies the stash there — clean,
  because it's the original base. Then merge/rebase that branch forward on your terms. This is the escape
  hatch for "my stash won't pop cleanly anymore."

- **`git stash clear` (and a wrong `drop`) is effectively unrecoverable — treat it as final.** A dropped
  stash becomes a dangling commit; it *can* sometimes be fished out via `git fsck --no-reflog`
  (that's `/oops` territory), but don't count on it. Never `clear` to "tidy up" unless you've confirmed
  every entry with `git stash list` + `show -p`.

- **Stashes are local and never pushed.** A stash lives only in your clone — it doesn't travel with
  `git push`, isn't shared, and won't survive a fresh clone. It's a short-term shelf, not a backup or a
  handoff. For anything you need to keep or share, make a real (even throwaway) commit or a branch.

## Gotchas

- **Untracked files are the #1 stash trap.** You stash, switch branch, and your shiny new files are
  *still there* (they were never stashed) or get in the way of the checkout. `git status --short` before
  stashing, and reach for `-u` the moment you see `??`.
- **`stash@{0}` is a moving target.** It's whatever's newest right now; every push/drop renumbers the
  stack. Re-run `git stash list` before referencing an index — don't trust a number you read a minute
  ago.
- **A stash with a conflict on pop is still on the stack.** Resolve, `git add`, then `git stash drop`
  manually. Assuming pop removed it leads to applying the same stash twice.
- **Stash ≠ worktree.** If the real need is "work two branches at once" or "keep my WIP live while I fix
  something else in parallel," that's `/worktree` — a second working directory, nothing shelved. Stash is
  for *parking* work to reach a clean tree on *this* checkout, briefly. If you find yourself stashing to
  bounce between branches repeatedly, switch to `/worktree`.
- **Don't stash as a backup.** It's local, unshared, and one `clear` from gone. For anything that
  matters beyond the next few minutes, commit it (`/lazy-commit`) or branch it.
