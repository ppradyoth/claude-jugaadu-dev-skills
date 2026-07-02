# Recipe: Pre-Push Safety Net

A two-minute ritual that catches the embarrassing stuff before it leaves your laptop.
Leaked tokens. Broken tests. A commit message that says "asdf". Run it before every push.

## When to use

Right before `git push`, especially on a branch headed for a PR. Cheap insurance.

## The chain

Run these skills back to back. With `/jugaadu` you can say it in one line:
> "secure check, run the tests, then write me a PR description"

Or run them manually:

1. **`/secure-diff`** — scan the branch diff for secrets and injection sinks.
   Stop here if it finds a blocker. A rotated key is a bad afternoon; a leaked one is a bad week.

2. **`/quick-test`** — make sure the thing you changed still has a passing test.
   If you touched logic and there's no test for it, this is the moment to add one.

3. **`/dep-audit`** — only if your diff touched `package.json`, `requirements.txt`, or a lockfile.
   New dependency? Confirm it's real and not a typosquat before it's in everyone's tree.

4. **`/pr-desc`** — generate the PR title and description from the branch diff.

## Manual fallback (no skills)

If you don't have the skills installed, the bare-metal version:

```bash
# 1. Look for obvious secrets in the diff
git diff main...HEAD | grep -nE 'AKIA[0-9A-Z]{16}|ghp_|sk-ant-|sk-[A-Za-z0-9]{20,}|-----BEGIN .*PRIVATE KEY-----'

# 2. Run the tests (swap for your runner)
npm test || pytest -q || go test ./...

# 3. Eyeball the commits you're about to push
git log main..HEAD --oneline
```

## Gotchas

- `grep` over a diff catches the *obvious* secret formats, not all of them. It's a tripwire, not a guarantee.
- A secret that's already in a previous commit won't show in `git diff main...HEAD`. If you suspect one
  landed earlier, scan history: `git log -p | grep -nE '...'` — and rotate anything you find.
- Tests passing locally ≠ tests passing in CI. Different env, different result. This catches the dumb breaks, not all breaks.
