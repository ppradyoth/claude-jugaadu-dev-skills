---
name: secure-diff
description: >
  Fast security pass over the current diff before you commit or push. Use when the user says
  "security check this", "review for security", "secure diff", "am I leaking anything", "scan
  before push", or invokes /secure-diff. Focuses on the things that actually bite: secrets,
  injection sinks, and unsafe handling of untrusted input. Not a replacement for a full audit.
---

# Secure Diff

A targeted security review of *what you're about to ship* — the staged or branch diff, not the
whole repo. Built by someone who red-teams AI systems for a living, so it knows where real
leaks hide, not just what a linter flags.

## Steps

1. Get the diff to review:
   ```bash
   git diff --cached            # staged changes (default)
   git diff main...HEAD         # whole branch, if nothing is staged
   ```
   If both are empty, say so and stop.

2. Scan for these in priority order. Report only what's actually present in the diff.

### 1. Secrets and credentials (highest priority)
Look for hardcoded values matching real key formats — don't guess, match patterns:
- AWS access key: `AKIA[0-9A-Z]{16}`
- GitHub PAT: `ghp_`, `github_pat_`
- OpenAI / Anthropic keys: `sk-`, `sk-ant-`
- Slack: `xox[baprs]-`
- Private keys: `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`
- Generic high-entropy strings assigned to `password`, `secret`, `token`, `api_key`
- A real value where a `${ENV_VAR}` reference should be

If you find one: flag it, and remind the user that committing then deleting is **not** enough —
the secret is in git history and must be rotated.

### 2. Injection sinks (untrusted input reaching a dangerous call)
- Shell: user/LLM input concatenated into `os.system`, `exec`, `subprocess(..., shell=True)`, backticks
- SQL: string-concatenated queries instead of parameterized ones
- Web: `innerHTML = userInput`, `dangerouslySetInnerHTML`, unescaped template output → XSS
- SSRF: a URL built from input passed straight to `fetch`/`requests`/`curl`
- Path traversal: user input used to build a filesystem path without normalization

### 3. AI / LLM-specific sinks (the stuff classic scanners miss)
- LLM output flowing straight into a sink above (`exec(llm_response)`, `innerHTML = completion`)
- External content (RAG docs, tool outputs, web pages) concatenated into a prompt with no
  data/instruction separation
- Tool/function arguments taken from model output without schema validation
- Missing human-in-the-loop on a high-impact action (send email, payment, file write, code run)

### 4. Footguns
- New dependency added — is it pinned? Is it a real, known package (typosquat check)?
- Debug flags, verbose logging of request bodies, `console.log` of tokens/PII
- `verify=False`, disabled TLS checks, `NODE_TLS_REJECT_UNAUTHORIZED=0`
- Overly broad CORS (`*`) or permissions

3. Report in this format:

```
## 🔴 Blockers (fix before pushing)
[file:line] — what's wrong, why it bites, the fix

## 🟡 Worth a look
[file:line] — concern + suggestion

## 🟢 Clean
Nothing critical in this diff. (List what you checked so the user trusts the pass.)
```

## Rules

- Review ONLY the diff — don't lecture about preexisting code outside the change
- Cite `file:line` for every finding so it's clickable and verifiable
- NEVER report a finding you can't point to in the diff — no theoretical "you might have"
- For a secret: rotate-then-remove, not remove-then-relax. Say it plainly.
- Rank by exploitability, not by how easy it is to spot
- If the diff is clean, say so confidently — don't manufacture findings to look thorough
- This is a fast pass, not a full audit. Say so if the change is security-critical enough to need one.
