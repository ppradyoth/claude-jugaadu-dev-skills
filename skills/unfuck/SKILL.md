---
name: unfuck
description: >
  Code is broken and you don't know why. This skill reads errors, recent changes,
  and project state to diagnose and fix. Use when the user says "unfuck this",
  "everything is broken", "it was working before", "help", "fix this", "why is this
  broken", or invokes /unfuck. Also trigger on expressions of frustration followed
  by an error or broken state description.
---

# Unfuck

Something broke. Find it. Fix it. Minimal words.

## Diagnosis Protocol

Run these in parallel where possible:

1. **What's the error?**
   - If the user pasted one, start there
   - If not, run the build/dev command and capture the error
   - Check `git status` for uncommitted changes

2. **What changed recently?**
   - `git diff` — unstaged changes
   - `git diff --cached` — staged changes
   - `git log --oneline -10` — recent commits
   - `git diff HEAD~3..HEAD` — last 3 commits' changes

3. **What's the environment?**
   - Node version, Python version, etc.
   - Are dependencies installed? (`node_modules` exists? `venv` active?)
   - Is the lock file in sync with the manifest?

4. **Common culprits — check these first:**
   - Dependency version mismatch (lock file out of sync)
   - Missing environment variable
   - Port already in use
   - Stale cache or build artifacts
   - Merge conflict markers left in code
   - Import path changed but not updated everywhere
   - Type error introduced by a recent refactor

## Fix Protocol

1. Identify the root cause — don't guess, trace it
2. Apply the minimal fix. Don't refactor. Don't clean up. Just fix.
3. Run the build/test again to verify
4. If it passes, output:
   ```
   **Cause**: [one sentence]
   **Fix**: [what was changed]
   **Verified**: [command that now passes]
   ```
5. If it still fails, repeat from step 1 with the new error

## Rules

- NEVER say "I see the issue" — just fix it
- NEVER suggest "have you tried restarting?" unless that's actually the fix
- NEVER do a clean install as first resort — diagnose first
- If the fix is "delete node_modules and reinstall", say WHY (lock file mismatch, corrupted install, etc.)
- If the user broke it with their own changes, be direct but not condescending
- If a dependency broke it, identify the exact version and pin or patch
- If it's a known bug, link the issue
- Maximum 2 rounds of diagnosis before asking the user for more context
- If you fix it, make the MINIMAL change. Don't sneak in refactors.
