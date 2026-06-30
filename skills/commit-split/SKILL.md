---
name: commit-split
description: >
  Take a messy working tree with many unrelated changes and split it into several clean,
  logical, individually-reviewable commits — one concern each. Use when the user says
  "split this commit", "break this up into commits", "these changes should be separate commits",
  "I forgot to commit incrementally", "untangle my working tree", or invokes /commit-split.
  Partitions an UNSTAGED blob by intent and stages each piece precisely. The companion to
  /lazy-commit (which writes ONE message for what's ALREADY staged).
---

# Commit Split

You sat down to fix one bug and got up three hours later with a feature, two refactors, a
formatting sweep, and an unrelated typo fix — all in one undivided pile of `git status`. Now
the diff is unreviewable and the history will be a lie. This skill untangles it.

`/lazy-commit` writes one message for changes you've *already staged*. This skill does the
hard part *before* that: it reads the whole pile, groups the changes by **intent**, and stages
each group on its own so every commit is one clean, reviewable concern. Think of it as the
sorting step; `/lazy-commit` is the writing step, and this skill calls that same message style
for each commit it makes.

The whole game: **partition by intent — feature vs fix vs refactor vs formatting vs the random
unrelated thing — so each commit tells one true story and (ideally) builds on its own.** Real
git only: `git add -p`, `git add <path>`, `git stash`, `git diff`. No rewriting history, no
magic.

## Steps

1. **Survey the whole pile first. Don't stage anything yet.**
   ```bash
   git status                  # tracked + untracked, the full picture
   git diff --stat             # which files, how big
   git diff                    # the actual hunks (unstaged)
   git diff --cached           # anything already staged? note it, don't clobber it
   ```
   Read every hunk before touching the index. You can't split what you haven't read.

2. **Back up before you touch anything.** One safety net so a mis-stage can never lose work:
   ```bash
   git stash push --keep-index --include-untracked -m "commit-split backup"
   git stash apply             # restore working tree, keep the stash as a fallback
   ```
   Keep that stash until the split is done and clean, then drop it. Never proceed without it.

3. **Cluster the hunks by concern.** Walk the diff and label each hunk with the *intent* it
   serves, not the file it lives in:
   - **feature** — new behavior
   - **fix** — corrects a bug
   - **refactor** — same behavior, different shape (rename, extract, move)
   - **formatting** — whitespace, import sort, reflow — *no logic change*
   - **unrelated** — the typo, the stray `console.log`, the thing you noticed in passing

   One file often spans several clusters. That's normal — that's why `git add -p` exists.

4. **For each cluster, stage exactly that and nothing else, then commit.** Pick the right tool
   per cluster:
   - Whole file belongs to one concern → `git add <path>`
   - File is split across concerns → `git add -p <path>` and `y`/`n` each hunk into *this*
     commit
   - A single hunk mixes two concerns → split it: `s` to break into smaller hunks, and if the
     lines are interleaved too tightly to separate, `e` to hand-edit the patch
   - A brand-new file → it's untracked; `git add -p` won't see it. `git add <path>` (or
     `git add -N <path>` first, then `git add -p` to stage it partially)

   Before committing each one, sanity-check the slice:
   ```bash
   git diff --cached --stat    # only this concern's files/hunks?
   ```
   Then commit it the **/lazy-commit way** — imperative subject under 72 chars, no period, body
   bullets if needed, matched to `git log --oneline -5` style:
   ```bash
   git commit -m "<message for this one concern>"
   ```
   Repeat for the next cluster.

5. **Keep formatting in its own commit, always.** A reformat mixed into a logic commit hides
   the real change in noise. If a hunk is pure whitespace/import-order, it goes in the
   formatting commit — never riding along with a fix.

6. **Verify nothing was left behind.**
   ```bash
   git status                  # clean, or only what you deliberately left unstaged
   git diff                    # should be empty if everything was committed
   ```
   If leftovers remain, either they're a cluster you haven't committed yet (do it) or you
   meant to leave them — say which.

7. **Show the result.** Drop the safety stash and print the new history so the user sees the
   split at a glance:
   ```bash
   git stash drop              # only after status is clean / intentional
   git log --oneline -<n>      # the n commits you just made, top to bottom
   ```

## Rules

- **Never lose a change.** The `git stash` backup in step 2 is non-negotiable — make it before
  any staging, drop it only once `git status` is clean or intentionally so.
- **Never force, never rewrite pushed history.** This splits *uncommitted* work into commits.
  No `--force`, no `reset --hard`, no amending what's already pushed.
- **One concern per commit.** If you can't describe a commit in a single clause without "and",
  it's two commits.
- **Formatting rides alone.** Whitespace/import/reflow changes get their own commit, every time.
- **Order so each commit builds.** Sequence dependencies first (the refactor before the feature
  that uses it). Don't produce an order where an intermediate commit won't compile if you can
  avoid it — but don't reorder *logic* in a way that changes behavior.
- **Default to acting, ask only on genuine ambiguity.** If two changes are clearly separate,
  just split them. Only stop and ask when a hunk truly could belong to either of two commits and
  it matters — then ask once, with the specific hunk, and proceed.
- **Reuse the commit-message bar.** Each message follows /lazy-commit conventions — don't write
  "split part 3", write what that commit actually does.

## Gotchas

- **`git add -p` controls:** `y` stage this hunk, `n` skip it, `s` split into smaller hunks,
  `e` manually edit the hunk to stage exact lines, `q` quit. `s` is your main tool; `e` is for
  when added and removed lines are interleaved in one hunk and `s` won't separate them.
- **Interleaved changes need `e`.** Two concerns on adjacent lines land in the same hunk and
  `s` can't split them. Edit the patch: keep the lines for *this* commit, delete the patch lines
  for the other (turn an unwanted `+` line into context by removing it from the patch; leave
  removals you don't want by prefixing the line correctly). Re-run for the second concern.
- **Untracked files are invisible to `-p`.** A new file won't show in `git add -p` until git
  knows about it. Use `git add <path>` for the whole file, or `git add -N <path>` to make it
  trackable and then `git add -p` to stage parts of it.
- **Already-staged changes.** If something was staged before you started (`git diff --cached`
  in step 1), decide deliberately whether it's its own commit or belongs with a cluster —
  don't let it silently merge into the first commit.
- **Whitespace-only noise.** `git diff -w` to see if a hunk is *only* whitespace before you
  decide it's a formatting hunk and not a logic change in disguise.
