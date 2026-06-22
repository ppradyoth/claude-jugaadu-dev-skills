---
name: tldr-error
description: >
  Diagnose an error in 2 lines and give the fix. No explanation, no context-setting,
  no "let me help you with that". Use when the user pastes an error, stack trace,
  exception, build failure, or says "wtf is this error", "fix this", "tldr error",
  or invokes /tldr-error. Also trigger on raw tracebacks pasted without any question.
---

# TLDR Error

The user pasted an error. They don't want a lecture. They want the fix.

## Response Format

```
**Problem**: [One sentence — what's actually wrong]
**Fix**: [The exact command, code change, or config edit to fix it]
```

That's it. Two lines. If the fix is a code change, show only the changed lines, not the whole file.

## Rules

- NEVER say "It looks like you're encountering..."
- NEVER explain what the error means — the user can read
- NEVER give multiple possible causes unless you genuinely can't tell (max 3, ranked by likelihood)
- NEVER suggest "try clearing your cache" unless that's actually the fix
- If the error is from a dependency, give the exact version pin or install command
- If the error is a typo, just say "typo" and show the fix
- If you need to see a file to diagnose, ask for that ONE file. Don't ask for "more context"
- If the fix requires multiple steps, number them. Keep each step to one line.
- If it's a known bug in a library, link the issue if you know it

## Common Patterns — Go Fast

- `ModuleNotFoundError` → `pip install X` or wrong venv
- `ENOENT` → file/dir doesn't exist, show the path that's wrong
- `CORS` → show the exact header or proxy config needed
- `TypeError: X is not a function` → wrong import or version mismatch
- `OOMKilled` → reduce batch size or increase memory limit
- Build failures → usually a missing dependency or wrong Node/Python version
