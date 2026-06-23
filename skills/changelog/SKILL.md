---
name: changelog
description: >
  Generate a clean, grouped changelog from git history between two refs. Use when the user
  says "write a changelog", "what changed since the last release", "changelog", "release notes",
  or invokes /changelog. Shell-first — git does the work, AI only groups and phrases.
---

# Changelog

Turn raw git history into release notes a human actually wants to read. Git does the heavy
lifting. AI only groups commits and rewrites cryptic messages into plain language.

## Steps

1. Find the range. If the user gave two refs, use them. Otherwise:
   ```bash
   git describe --tags --abbrev=0 2>/dev/null   # most recent tag
   git tag --sort=-creatordate | head -5         # recent tags to pick from
   ```
   Default range: last tag → `HEAD`. If there are no tags, use the first commit → `HEAD`.

2. Pull the commits in the range:
   ```bash
   git log <from>..<to> --no-merges --pretty=format:'%h%x09%s%x09%an'
   ```

3. Group commits by type. Detect the type from Conventional Commit prefixes when present
   (`feat:`, `fix:`, `perf:`, `refactor:`, `docs:`, `test:`, `chore:`, `build:`, `ci:`).
   If the repo doesn't use prefixes, infer the group from the message verb.

   Map to reader-facing sections:
   - `feat` → **Added**
   - `fix` → **Fixed**
   - `perf` → **Performance**
   - `refactor` / `build` / `ci` / `chore` → **Changed** (collapse if noisy)
   - `docs` → **Docs**
   - Breaking changes (`!` in prefix, or `BREAKING CHANGE` in body) → **⚠ Breaking** at the top

4. Rewrite each line for a reader, not a committer:
   - Drop the prefix and the scope noise (`fix(api):` → just the change)
   - Present tense, no trailing period
   - Keep the short hash in parentheses at the end for traceability

5. Output in Keep a Changelog format:

   ```markdown
   ## [<version or "Unreleased">] — <YYYY-MM-DD>

   ### ⚠ Breaking
   - ...

   ### Added
   - New thing users can now do (a1b2c3d)

   ### Fixed
   - Bug that no longer bites (e4f5g6h)
   ```

## Rules

- NEVER invent a change that isn't in the git log — every bullet maps to a real commit
- Collapse pure-noise commits (`chore: maintenance update`, `wip`, `fixup`) into one line or drop them
- If a commit message is useless ("update", "stuff"), read the diff for that commit before phrasing it
- Don't include merge commits — they're plumbing, not changes
- If nothing user-facing changed, say so in one line instead of padding the list
- Match the version header to the tag you're cutting; use "Unreleased" if there's no tag yet
