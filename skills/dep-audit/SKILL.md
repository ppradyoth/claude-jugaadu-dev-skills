---
name: dep-audit
description: >
  Check for outdated, vulnerable, or unused dependencies using CLI tools, not AI tokens.
  Use when the user says "check deps", "audit dependencies", "are my packages outdated",
  "security audit", "dep audit", or invokes /dep-audit. This is a shell-first skill —
  AI is only used to summarize results, not to do the checking.
---

# Dep Audit

Audit dependencies using CLI tools. Zero AI tokens for the actual checking — only for summarizing.

## Steps

1. Detect the package manager:
   - `package-lock.json` / `yarn.lock` / `pnpm-lock.yaml` → Node
   - `pyproject.toml` / `requirements.txt` / `Pipfile` → Python
   - `go.sum` → Go
   - `Cargo.lock` → Rust
   - `Gemfile.lock` → Ruby

2. Run the appropriate audit commands:

### Node (npm/yarn/pnpm)
```bash
npm audit --json 2>/dev/null || true
npm outdated --json 2>/dev/null || true
npx depcheck --json 2>/dev/null || true
```

### Python
```bash
pip audit --format=json 2>/dev/null || safety check --json 2>/dev/null || true
pip list --outdated --format=json 2>/dev/null || true
```

### Go
```bash
go list -m -u all 2>/dev/null || true
govulncheck ./... 2>/dev/null || true
```

### Rust
```bash
cargo audit --json 2>/dev/null || true
cargo outdated --format json 2>/dev/null || true
```

3. Summarize results in this format:

```
## Security Vulnerabilities
[severity] package@version — description (fix: upgrade to X)

## Outdated (Major)
package: current → latest (breaking changes likely)

## Outdated (Minor/Patch)
package: current → latest

## Unused Dependencies
package — not imported anywhere, safe to remove

## Verdict
[one sentence: "3 critical vulns to fix now, 5 outdated packages, 2 unused deps to remove"]
```

## Rules

- NEVER manually check if a package is outdated — use the CLI tools
- NEVER guess at vulnerabilities — only report what the audit tools find
- If an audit tool isn't installed, tell the user to install it (one command) and stop
- Prioritize: security vulns > major outdated > unused deps > minor outdated
- For security vulns, include the CVE number if available
- For breaking changes, note what might break if upgraded
- Don't recommend upgrading everything — prioritize what matters
