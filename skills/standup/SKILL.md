---
name: standup
description: >
  Generate your daily standup update from git history — what you actually shipped, not what
  you half-remember. Use when the user says "standup", "what did I do yesterday", "daily update",
  "status update for the team", or invokes /standup. Git does the work; the AI only groups and
  phrases. Reports only real commits — never invents progress.
---

# Standup

Your standup, reconstructed from the commits you actually made. No "uhh, let me think." Git
remembers; this just reads it back in the shape your team wants.

The work is done by git. The model only groups related commits and phrases them like a human —
so this stays cheap and never makes up work that isn't in the log.

## Steps

1. Figure out **who** you are and **since when** to look:
   ```bash
   git config user.email                  # the author to filter by
   ```
   - Default window: since the last working day. On a Monday that's the previous Friday
     (`--since="last friday"`); otherwise `--since="yesterday 6am"`.
   - If the user names a window ("since Monday", "this week"), use that instead.

2. Pull **what shipped** in that window, by this author, on the current repo:
   ```bash
   git log --since="yesterday 6am" --author="$(git config user.email)" \
     --no-merges --pretty=format:'%h %s' --stat
   ```
   - If the user works across several repos, run the same `git log` in each path they give and
     label the output by repo.
   - If nothing comes back, say so plainly ("no commits in this window") — don't pad it.

3. Pull **what's in flight** (this becomes "Today", grounded in real state, not a guess):
   ```bash
   git branch --show-current                       # what you're on
   git status --short                              # uncommitted work in progress
   git log --oneline @{push}..HEAD 2>/dev/null     # committed but not yet pushed
   ```

4. Group the commits into themes (feature, fix, refactor, docs, tests) and write the update in
   this shape:

   ```
   *Yesterday* (or the window you used)
   - <plain-English summary of a theme> (`abc1234`, `def5678`)
   - <next theme> (`...`)

   *Today*
   - <derived from the in-flight branch / uncommitted work — what it's clearly leading to>
   - (if nothing is in flight: leave a single "— planning next" line, don't invent tasks)

   *Blockers*
   - None. (Only list a blocker if the user actually mentions one — never manufacture one.)
   ```

5. Keep commit hashes next to each line so anyone can click through and verify. Output the
   update and nothing else — no preamble, no "here's your standup."

## Rules

- Report ONLY commits that appear in the `git log` output. If it's not in the log, it didn't
  happen — don't infer it.
- "Today" is derived from real in-flight state (current branch name, unstaged work,
  unpushed commits), not imagined. If there's genuinely nothing in flight, say "planning next"
  rather than fabricating a task list.
- NEVER invent a blocker. Blockers come from the user, not from you.
- Collapse noise: a dozen "fix typo" commits become one line, not twelve.
- Match the team's channel: default to Slack-style (`*bold*`, short bullets). If the user wants
  plain text or markdown, switch.
- Keep it to what a person would actually say standing up — three sections, a handful of bullets,
  done.
- Don't editorialize on how hard the work was or how much got done. Just report it.
