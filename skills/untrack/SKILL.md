---
name: untrack
description: >
  Find files that are committed but never should have been — a `.env`, `node_modules/`,
  build output, `.DS_Store`, an IDE folder — and stop tracking them without deleting your
  local copy. Use when the user says "I committed my .env", "stop tracking this file",
  "this shouldn't be in git", "untrack node_modules", "clean up what's committed", "my
  gitignore isn't working", or invokes /untrack. Adds the pattern to `.gitignore`,
  `git rm --cached`s the file, and — for anything secret — tells you the untrack is not
  the fix, rotation is.
---

# Untrack

`.gitignore` only ignores files git *isn't already tracking*. The moment a file is committed,
the ignore rule stops applying to it — git keeps updating it forever, and every `git status`
that looks clean is quietly lying. So the `.env` you added on day one keeps shipping its keys to
the remote, and adding `.env` to `.gitignore` on day fifty does nothing.

This skill finds the files that are tracked but shouldn't be, stops tracking them (keeping your
local copy), and adds the pattern to `.gitignore` so it doesn't come back.

The whole game: **list what git tracks, match it against the things nobody commits on purpose,
then `git rm --cached` (not `rm`) the confirmed ones and ignore the pattern — while being loud
that untracking a secret is not the same as removing it.**

## Steps

1. **List everything git actually tracks.** Not the working tree — the index:
   ```bash
   git ls-files
   ```
   This is the source of truth. A file here is tracked no matter what `.gitignore` says.

2. **Match against the never-commit patterns.** Flag tracked files that fall into these buckets:
   - **Secrets / credentials** (highest priority): `.env`, `.env.*` (but not `.env.example`/`.env.sample`/`.env.template`), `*.pem`, `*.key`, `id_rsa`, `*.p12`, `*.keystore`, `credentials`, `.aws/credentials`, `*.pfx`.
   - **Dependencies / vendored**: `node_modules/`, `vendor/`, `.venv/`, `venv/`, `bower_components/`.
   - **Build output**: `dist/`, `build/`, `target/`, `out/`, `*.pyc`, `__pycache__/`, `.next/`, `coverage/`.
   - **OS / editor cruft**: `.DS_Store`, `Thumbs.db`, `.idea/`, `.vscode/` (ask — some teams commit it on purpose), `*.swp`.
   - **Logs / local state**: `*.log`, `.terraform/`, `*.tfstate`, `*.sqlite`, `*.db` (ask — some repos ship a seed DB).

   Grep the tracked list rather than the filesystem so you only ever consider what's committed:
   ```bash
   git ls-files | grep -iE '(^|/)\.env($|\.)|\.(pem|key|p12|keystore|pfx)$|(^|/)node_modules/|(^|/)(dist|build|target|out|coverage)/|\.DS_Store$|(^|/)__pycache__/'
   ```
   Tune the pattern to what you actually find; don't blind-run it and assume the match list is complete.

3. **Separate "definitely" from "ask first".** A committed `.env` or `*.key` is unambiguous — it
   should never have been tracked. But `.vscode/`, a seed `*.db`, or a checked-in `dist/` can be
   deliberate (some projects vendor their build, some ship editor settings for the whole team).
   Never untrack an "ask first" file without confirming it isn't intentional.

4. **Show the list, grouped, before touching anything.** Put secrets at the top and name the
   consequence:
   > "`config/.env` and `deploy/prod.key` are tracked — these are secrets and have been in the
   > repo since they were committed. 4 more are cruft (`.DS_Store`, `dist/`, `node_modules/`).
   > Untrack all of them (keeping your local copies)?"

5. **Untrack with `--cached` — never a plain `rm`.** This removes the file from git's index but
   leaves it on disk exactly as-is:
   ```bash
   git rm --cached <file>            # one file
   git rm -r --cached node_modules/  # a directory
   ```
   `--cached` is the whole point: without it, `git rm` deletes your working copy too. For a
   directory, `-r` is required. Nothing is committed yet — the removal is staged.

6. **Add the pattern to `.gitignore` so it can't creep back.** Untracking without ignoring just
   means the next `git add .` re-commits it. Append the *pattern*, not the one path, when the
   pattern is the right scope (`node_modules/`, not `node_modules/left-pad/...`). De-dupe against
   what's already there; don't append a rule that's already present.

7. **Commit the two changes together and say what's still exposed.**
   ```bash
   git commit -m "Stop tracking <thing>; add to .gitignore"
   ```
   Then state plainly which files were secrets, because untracking them going forward does **not**
   undo the exposure (next rule).

## Rules

- **`git rm --cached`, never `git rm`.** The `--cached` flag is the entire safety property: it
  untracks the file and keeps it on disk. A bare `git rm` deletes the working copy — losing the
  user's real `.env`. If you catch yourself about to run `git rm` without `--cached`, stop.

- **Untracking a secret is not the fix — rotation is.** This is the one that matters. Once a key,
  token, or password has been committed and pushed, it lives in the repo's history forever;
  anyone who cloned or forked already has it. `git rm --cached` only stops *future* commits from
  carrying it. The real remediation is: **rotate/revoke the credential** (assume it's compromised),
  and only then, if the history itself must be scrubbed, use `git filter-repo` or the GitHub
  support process for a force-rewrite — a heavy, separate operation you should call out but not
  run reflexively. Never let the user believe an untrack made a leaked key safe.

- **`.gitignore` alone would have done nothing here.** Say it out loud when relevant — the reason
  the file kept showing up despite being "ignored" is that it was tracked before the rule existed.
  Ignore rules are not retroactive. That's why this skill exists and a `.gitignore` edit doesn't.

- **Ask before untracking anything that might be deliberate.** `dist/`, `.vscode/`, a seed
  `*.db`, vendored `vendor/` — plenty of real repos commit these on purpose. Untrack the
  unambiguous cruft freely; confirm the judgment calls. When unsure, leave it and ask.

- **Ignore the pattern, not the path — but only when the pattern is right.** For `node_modules/`
  the directory pattern is correct. For a single stray `report-final.xlsx`, ignore that path, not
  `*.xlsx`, which would swallow files the user *does* want tracked. Match the rule's blast radius
  to the intent.

- **Don't touch `.env.example` and friends.** Committed template files (`.env.example`,
  `.env.sample`, `.env.template`) are supposed to be tracked — they document the shape without the
  values. Exclude them from the secrets match; untracking them removes useful docs.

- **Local only, one repo.** This operates on the current repo's index. It never rewrites history
  and never pushes a `--delete` — the loud, dangerous parts are called out for the user to do
  deliberately, not run as a side effect of a cleanup.
