---
name: pr-desc
description: >
  Generate a PR title and description from the current branch diff. Use when the user
  says "write PR description", "pr desc", "create PR", "describe this PR", or invokes
  /pr-desc. Also trigger when the user is about to push and wants a PR created.
---

# PR Desc

Generate a PR title + description from the branch diff. No conversation.

## Steps

1. Detect the base branch: `git rev-parse --abbrev-ref HEAD` and find the merge base with main/master/develop
2. Get the full diff: `git diff main...HEAD` (or the appropriate base)
3. Get commit list: `git log --oneline main..HEAD`
4. Check for existing PR templates: `.github/pull_request_template.md`
5. Generate the PR description

## Output Format

If the repo has a PR template, fill it in. Otherwise use:

```markdown
## Summary
[2-3 bullet points — what changed and why]

## Changes
[Bulleted list of specific changes, grouped by area if touching multiple parts]

## Testing
[How to verify — commands to run, pages to check, or "covered by existing tests"]
```

## Rules

- Title: imperative mood, under 70 chars, specific (not "Update code" or "Fix stuff")
- NEVER list every file changed — group by logical change
- NEVER include the diff itself in the description
- NEVER write "This PR..." — just describe what it does
- If there's only one commit, the PR description can be minimal
- If there are many commits, synthesize — don't just list them
- Mention breaking changes prominently if any
- If the user says "create PR", actually create it with `gh pr create`
- If the user just wants the description, output it as copyable text

## Smart Defaults

- If the branch name contains an issue number (e.g., `fix/123-login-bug`), reference it: "Fixes #123"
- If the diff touches tests, mention what's covered under Testing
- If the diff adds a dependency, mention it and why
- If the diff deletes more than it adds, lead with what was removed and why
