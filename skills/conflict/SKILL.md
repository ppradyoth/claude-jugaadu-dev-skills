---
name: conflict
description: >
  Resolve git merge conflicts safely and correctly — during a merge, rebase, cherry-pick, or stash
  pop. Use when the user says "I have merge conflicts", "fix the conflicts", "resolve this rebase",
  "git says CONFLICT", "help me merge", or invokes /conflict. Reads each conflict hunk and BOTH
  sides' history before resolving; never blindly picks a side, never marks resolved without the user
  confirming intent on any hunk where both sides changed real logic.
---

# Conflict

A merge, rebase, or cherry-pick just stopped with `CONFLICT`. The working tree has files full of
`<<<<<<<`, `=======`, `>>>>>>>` markers, and the instinct is to delete the markers as fast as
possible and move on. That's how you silently drop someone's bug fix or reintroduce a deleted
line.

A conflict marker is not noise to clear — it's git telling you **two changes touched the same place
and it can't decide which intent wins.** Your job is to understand both intents and produce the
version that keeps both where possible, picks deliberately where not, and never guesses on logic.

The whole game is: **know which operation you're in, read both sides' reasoning, resolve each hunk
on its merits, then verify the result actually builds before you continue the operation.**

## Steps

1. **Figure out which operation conflicted — the resolution differs.** "ours" vs "theirs" flips
   between merge and rebase, and that flip is the #1 way people resolve conflicts backwards.
   ```bash
   git status                      # tells you: merging / rebasing / cherry-picking / reverting
   ```
   - **Merge:** `ours` = the branch you're on (your target, e.g. `main`); `theirs` = the branch
     being merged in.
   - **Rebase / cherry-pick:** it's inverted — `ours` = the commit being replayed *onto* (the
     upstream you're rebasing onto), `theirs` = your commit being replayed. Re-read that until it's
     not surprising, because the labels feel backwards during a rebase.

2. **List the battlefield.** See exactly which files conflict before opening anything.
   ```bash
   git diff --name-only --diff-filter=U     # only the unmerged (conflicted) files
   ```

3. **For each conflicted file, read both sides AND why they changed.** Don't resolve from the hunk
   alone — pull up the intent behind each side.
   ```bash
   git log --merge -p -- <file>     # the commits from each side that touch this file
   ```
   The markers split each hunk into: `<<<<<<< ours` … `=======` … `>>>>>>> theirs`. Ask of every
   hunk: *are these two changes doing the same thing two ways (pick one), or two different things
   in the same spot (keep both)?*

4. **Resolve each hunk on its merits — not by a blanket rule.**
   - **Both sides did different, compatible things** (e.g. each added a different import, function,
     or config key) → **keep both**, ordered sensibly. This is the most common real case and the
     one "pick a side" gets wrong.
   - **Same change, two phrasings** → keep the clearer one.
   - **One side is a deliberate revert/delete of what the other added** → this needs the user.
     Don't silently resurrect deleted code or drop a fix. Ask which intent is current.
   - **Genuinely can't tell** → ask one specific question naming the file and the competing
     intents. Never coin-flip on logic.
   - For a file you're certain belongs entirely to one side (rare — e.g. a generated lockfile),
     `git checkout --ours <file>` / `--theirs <file>` is legitimate. Name it out loud first so the
     wholesale choice is explicit.

5. **Remove every marker and stage the resolved files.** A leftover `=======` compiles in some
   languages and ships a corrupt file. Sweep for them before staging.
   ```bash
   git diff --check                 # flags leftover conflict markers
   git add <resolved-file>
   ```

6. **Verify the resolution actually works — before continuing the operation.** A resolution that
   parses but is logically wrong is worse than the conflict. Build/lint/test the touched area.
   ```bash
   # whatever the repo uses:
   npm test || pytest -q || go build ./... || cargo check
   ```

7. **Continue the operation, don't re-commit blindly.** Use the operation's own continue command —
   it preserves authorship and the rest of the rebase/merge sequence.
   ```bash
   git merge --continue        # or: git rebase --continue / git cherry-pick --continue
   ```
   For a rebase, conflicts can recur on the *next* replayed commit — repeat steps 3–7 until
   `git status` is clean. State that more conflicts may follow so it's not a surprise.

## Rules

- **Identify the operation first.** Resolving a rebase with merge's ours/theirs mental model
  inverts every wholesale choice. `git status` tells you which one you're in — read it before
  touching a marker.
- **Read both sides' history, never resolve from the hunk alone.** `git log --merge -p` shows the
  *why*. A hunk without its intent is a guess.
- **Keep-both is the default for compatible changes.** Most conflicts are two people editing near
  each other, not fighting over the same line. Picking a side there silently deletes real work.
- **Never silently drop a fix or resurrect deleted code.** If one side deleted what the other
  changed, that's a decision for the user — surface it, don't paper over it.
- **No coin-flips on logic.** If you can't determine the correct merge, ask one specific question
  naming the file and the two intents. "I picked theirs" is not a resolution strategy.
- **Sweep for leftover markers before staging.** `git diff --check`. A surviving `<<<<<<<` /
  `=======` / `>>>>>>>` is a broken file, not a resolved one.
- **Verify before continuing.** Build/test the resolved area. A clean `git status` proves the
  markers are gone, not that the merge is correct.
- **`--continue`, never a fresh commit.** Let git finish the merge/rebase/cherry-pick on its own
  terms so authorship and sequence stay intact.
- **Bail-out is always available.** If the resolution is going sideways, `git merge --abort` /
  `git rebase --abort` returns to the pre-operation state cleanly. Offer it before forcing a bad
  resolution — and if work was lost in an earlier botched attempt, that's a job for `/oops`.
