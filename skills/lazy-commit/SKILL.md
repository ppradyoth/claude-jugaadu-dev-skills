---
name: lazy-commit
description: >
  Generate a git commit message from staged changes and commit. One shot, no chat.
  Use when the user says "commit this", "lazy commit", "just commit", "write commit message",
  or invokes /lazy-commit. Never ask follow-up questions — just read the diff and commit.
---

# Lazy Commit

Generate a commit message and commit. No conversation. No confirmation. No explanation.

## Steps

1. Run `git diff --cached --stat` and `git diff --cached` (staged changes only)
2. If nothing staged, run `git diff --stat` and tell the user to stage first. Stop.
3. Read `git log --oneline -5` to match the repo's commit style
4. Write the commit message:
   - First line: imperative mood, under 72 chars, no period
   - Describe WHAT changed and WHY, not HOW
   - If multiple logical changes, use bullet points in body
   - Match the existing commit style from step 3
5. Run `git commit -m "<message>"`
6. Output the commit hash and one-line summary. Nothing else.

## Rules

- NEVER ask "does this look good?" — just commit
- NEVER explain what the diff contains — the user wrote it, they know
- NEVER add `Co-Authored-By` unless the user asks
- If the diff is trivial (typo, formatting), the message should be trivial too
- If the diff touches tests, say what's being tested, not "add tests"
- No emoji in commit messages unless the repo already uses them
