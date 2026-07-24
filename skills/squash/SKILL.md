---
name: squash
description: >
  Collapse a messy feature branch — twelve "wip", "fix", "oops typo", "actually fix" commits —
  into one clean commit (or a few logical ones) before you open or merge a PR. Use when the user
  says "squash this branch", "clean up my commits", "my history is a mess", "collapse these wip
  commits", "one commit before merge", or invokes /squash. The inverse of /commit-split
  (which turns one pile into many); this turns many into few. Non-interactive: uses
  `git reset --soft` to the merge-base, never a hand-driven `rebase -i`. Backs up the branch
  first and pushes with `--force-with-lease`, never a bare `--force`.
---

# Squash

Your feature branch works. The history is embarrassing. `wip`, `wip2`, `fix test`, `oops`,
`actually fix test`, `revert`, `pls`. Nobody reviewing the PR wants to read that, and once it
merges it's in `main` forever.

This skill flattens that branch into **one clean commit — or a small number of logical ones —
without an interactive rebase.** `/commit-split` takes one messy blob and *fans it out* into
many clean commits; `/squash` does the opposite: it takes many junk commits and *collapses*
them into the few that tell the real story. Same goal — honest, reviewable history — from the
other direction.

The mechanism is deliberately boring and safe: reset the branch pointer back to where it
diverged, keep every change staged, and re-commit cleanly. No `rebase -i`, no reordering, no
per-commit conflict marathon. The *tree* never changes — only the commit history on top of it.

**The whole game: `git reset --soft <merge-base>` keeps all your work staged while throwing
away only the messy commit *messages* — then you write the commit history you wish you'd made.**

## Preconditions — check these first, bail if they fail

Squashing rewrites history. That is safe on a private feature branch and dangerous on a shared
one. Before touching anything:

1. **You're on a feature branch, not the trunk.**
   ```bash
   git branch --show-current          # must NOT be main / master / develop
   ```
   Refuse to squash `main`/`master`/`develop`/`release/*`. Say why and stop.

2. **The branch isn't already merged, and others probably aren't building on it.** A squash
   force-pushes; if a teammate has these commits checked out, you'll wreck their branch. If the
   PR is already merged, don't squash — start fresh work instead. When unsure whether it's
   shared, ask once before rewriting.

3. **The working tree is clean** (commit or stash first). A soft reset onto a dirty tree mixes
   uncommitted work into the squash silently.
   ```bash
   git status --porcelain             # expect empty
   ```

## Steps

1. **Find where this branch diverged from its base.** Everything *after* the merge-base is
   yours to squash; everything at or before it is shared history you must not touch.
   ```bash
   git fetch origin                                   # so the base ref is current
   BASE=$(git merge-base HEAD origin/main)            # swap origin/main for the real base branch
   git log --oneline "$BASE"..HEAD                    # the exact commits about to collapse
   git diff --stat "$BASE"..HEAD                      # the net change they add up to
   ```
   Show the user that commit list and the net diffstat. This is what "before" looks like.

2. **Back up the branch before you rewrite it.** One command, and a bad squash is a one-line undo.
   ```bash
   git branch backup/$(git branch --show-current)     # a real ref at the current tip
   ```
   Keep it until the squash is verified good. If anything goes wrong:
   `git reset --hard backup/<branch>` puts you exactly back.

3. **Soft-reset to the base. This is the squash.** `--soft` moves only the branch pointer; it
   leaves the working tree untouched and every change from those commits **staged**.
   ```bash
   git reset --soft "$BASE"
   git status                                         # all your work, staged, as one set
   git diff --cached --stat                           # identical to the step-1 diffstat — verify it
   ```
   That diffstat **must match** step 1's. If it doesn't, something outside the branch got pulled
   in — `git reset --hard backup/<branch>` and reassess. Do not commit a mismatch.

4. **Re-commit as the history you wish you'd made.** Two honest options — pick by what the diff
   actually contains:
   - **One concern → one commit** (the common case):
     ```bash
     git commit -m "<clear imperative subject>" -m "<why, if it needs saying>"
     ```
     Write it the `/lazy-commit` way: imperative subject under 72 chars, no trailing period,
     matched to the repo's `git log --oneline -5` style.
   - **Genuinely separate concerns in one branch** (e.g. a refactor *and* the feature that uses
     it): don't force them into one lie. Everything is staged right now, so hand off to
     **`/commit-split`** — partition the staged pile by intent and make several clean commits.
     Squash-to-few, not always squash-to-one.

5. **Preserve co-authorship.** Squashing drops the individual commits, so any `Co-authored-by:`
   trailers on them vanish unless you carry them forward. Check before you flatten:
   ```bash
   git log "$BASE"..HEAD --format='%(trailers:key=Co-authored-by)' | sort -u
   ```
   Add every distinct collaborator back as a `Co-authored-by:` trailer in the new commit's body
   (blank line, then one trailer per line). Don't erase someone's credit in a cleanup.

6. **Verify the tree is byte-identical to before.** The proof a squash was lossless: the code
   didn't move, only the history did.
   ```bash
   git diff backup/$(git branch --show-current) --stat   # MUST be empty — same tree, new history
   ```
   Empty output = the squash changed nothing but the commit list. Non-empty = you lost or gained
   changes; reset to the backup and try again. If tests are quick, run them now too.

7. **Publish with `--force-with-lease`, never bare `--force`.** The branch history changed, so a
   plain push is rejected — that rejection is a safety feature, don't defeat it with `--force`.
   ```bash
   git push --force-with-lease                            # refuses if the remote moved under you
   ```
   `--force-with-lease` aborts if someone pushed to the branch since your last fetch (bare
   `--force` would silently overwrite their work). If it's rejected: **stop** — someone else is
   on this branch. Re-fetch, and don't rewrite shared commits without asking.

8. **Drop the backup once it's verified good.**
   ```bash
   git branch -D backup/<branch>          # only after step 6 was clean and the push succeeded
   git log --oneline "$BASE"..HEAD        # the clean "after" — show it
   ```

## Rules

- **Never squash the trunk or a shared branch.** `main`/`master`/`develop`/`release/*` are
  off-limits. If others may have the branch checked out, ask before rewriting — a force-push
  rewrites their history too.
- **Back up before you rewrite, every time.** The `backup/<branch>` ref in step 2 is
  non-negotiable; delete it only after the tree-identical check passes and the push lands.
- **The tree must not change.** Step 6's `git diff backup/... --stat` being empty is the
  contract. A squash that alters the diff is a bug — reset and retry, never ship it.
- **`--force-with-lease`, never `--force`.** If the lease push is rejected, someone else moved
  the branch. Stop and coordinate; don't overwrite them.
- **Don't invent a story.** The squash message describes what the branch *actually* does end to
  end — read the net diff, don't paste "wip" or "squash 5 commits".
- **Squash to *few*, not blindly to *one*.** If the branch legitimately contains two unrelated
  concerns, split them (hand off to `/commit-split`) instead of burying one inside the other.
- **Keep co-authors.** Carry every `Co-authored-by:` from the collapsed commits into the new one.
- **Real git only.** `reset --soft`, `merge-base`, `commit`, `push --force-with-lease`. No
  history surgery beyond collapsing your own unmerged commits.

## Gotchas

- **Why `reset --soft` beats `rebase -i` for an agent.** Interactive rebase needs a terminal
  editor and replays commits one-by-one (each a possible conflict). `reset --soft <base>`
  collapses everything in a single non-interactive step with **zero** conflicts — the tree never
  moves, so there's nothing to re-merge. It's the right tool when the goal is "one/few commits at
  the tip", not "reorder or edit specific old commits".
- **Get the base right, or you'll squash too much.** `git merge-base HEAD origin/main` is the
  divergence point. If you hardcode the wrong base (or use a stale local `main`), you can sweep
  commits that aren't yours into the reset. Always `git fetch` first and eyeball the step-1 log.
- **Merge commits inside the branch.** If you merged `main` into your branch midway, a soft reset
  to the merge-base flattens those merges away — usually what you want, but the net diff now
  includes whatever you pulled in from `main` after the base. Re-check step 6; if the diffstat
  looks too big, your base is older than you think.
- **Already pushed + PR open?** Fine to squash *your own* PR branch — reviewers generally prefer
  it. The force-with-lease updates the PR in place. Just never do it mid-review without a heads-up
  if others pushed to it.
- **Already merged?** Don't squash after merge — the commits are in `main` now and rewriting them
  is trunk surgery. Treat follow-ups as new commits, not a rewrite.
- **Signed commits.** A soft reset + new commit re-signs with your current config; the old
  per-commit signatures are gone (they signed commits that no longer exist). Expected, not a bug.
