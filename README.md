# Claude Jugaadu Dev Skills 🛠️

> *Jugaadu* (जुगाड़ू) — Hindi for clever, resourceful, hacky-in-the-best-way.

> *Jugaadu* (ಜುಗಾಡು) — Kannada for the same vibe. Because jugaad has no language barrier.

**A collection of custom [Claude Code](https://claude.com/claude-code) skills that delete the boring parts of your day.**

Commit messages you didn't write. Security scans before every push. PR descriptions out of thin air. Each one is a single folder you drop into `~/.claude/skills/` — no setup, no config, no SaaS.

<p>
  <img src="https://img.shields.io/github/stars/ppradyoth/claude-jugaadu-dev-skills?style=flat-square&color=cc2200" alt="Stars" />
  <img src="https://img.shields.io/github/last-commit/ppradyoth/claude-jugaadu-dev-skills?style=flat-square&color=0e75b6" alt="Last commit" />
  <img src="https://img.shields.io/github/license/ppradyoth/claude-jugaadu-dev-skills?style=flat-square" alt="License" />
  <img src="https://img.shields.io/badge/Built%20with-Claude%20Code-cc2200?style=flat-square&logo=anthropic&logoColor=white" alt="Built with Claude Code" />
</p>

---

## Contents

- [Why this exists](#why-this-exists)
- [Quick start](#quick-start)
- [Skills gallery](#skills-gallery)
- [Recipes](#recipes)
- [Prompt templates](#prompt-templates)
- [Repo layout](#repo-layout)
- [Philosophy](#philosophy)
- [Contributing](#contributing)
- [License](#license)

---

## Why this exists

Claude Code is great at the hard parts. It's wasteful on the boring parts.

Writing a commit message. Drafting a PR description. Checking a diff for a leaked token. None of these are hard — they're just friction. And friction is where your day leaks away, ten minutes at a time.

A *skill* is a tiny markdown file that teaches Claude Code to do one of these chores the same way every time, in one shot, with no back-and-forth. No menus. No "does this look good?" Just the thing, done.

This repo is the collection I actually use. Each skill is self-contained and shareable — copy one folder, get one superpower. Star the repo and you've got the whole set in arm's reach.

## Quick start

Skills are folders. Claude Code reads them from `~/.claude/skills/` (yours, everywhere) or a project's `.claude/skills/` (that repo only). [[docs](https://code.claude.com/docs/en/skills)]

**Grab one skill:**

```bash
# Clone, then copy the one you want into your personal skills dir
git clone https://github.com/ppradyoth/claude-jugaadu-dev-skills.git
mkdir -p ~/.claude/skills
cp -r claude-jugaadu-dev-skills/skills/lazy-commit ~/.claude/skills/
```

**Grab all of them:**

```bash
git clone https://github.com/ppradyoth/claude-jugaadu-dev-skills.git
mkdir -p ~/.claude/skills
cp -r claude-jugaadu-dev-skills/skills/* ~/.claude/skills/
```

Claude Code picks them up live — no restart. Then just talk:

> "commit this" → `/lazy-commit` fires
> "security check before I push" → `/secure-diff` fires
> "fix this and write me a PR" → `/jugaadu` chains `/unfuck` → `/pr-desc`

That's it. No API keys, no install step, no dependencies. The skill is the file.

## Skills gallery

### 🧠 Master skill
| Skill | What it does |
|-------|-------------|
| [`/jugaadu`](skills/jugaadu/) | The brain. Detects what you need and auto-routes to the right skill — or chains several in one shot. Say "fix this and commit" and it runs `/unfuck` → `/lazy-commit`. No menus, no questions. |

### 🔒 Security & safety
| Skill | What it does |
|-------|-------------|
| [`/secure-diff`](skills/secure-diff/) | Fast security pass over the diff you're about to push. Secrets, injection sinks, and the AI-specific leaks classic scanners miss. Built by someone who red-teams AI systems for a living. |
| [`/prompt-injection-scan`](skills/prompt-injection-scan/) | Repo-wide hunt for the #1 LLM bug (OWASP LLM01): untrusted input reaching a model prompt with no guardrail. Greps the sinks, traces the taint, tells you which are real. Catches indirect injection — poisoned RAG docs, tool results — that most scanners miss. `/secure-diff` for a diff; this for the whole codebase. |
| [`/dep-audit`](skills/dep-audit/) | Audit deps for vulnerabilities, outdated packages, and unused imports. Shell-first — CLI does the checking, AI only summarizes. Near-zero tokens. |
| [`/untrack`](skills/untrack/) | Committed a `.env`? `.gitignore` won't help — it only ignores files git *isn't already tracking*. Finds tracked files that should never have been (secrets, `node_modules/`, build output, `.DS_Store`), `git rm --cached`s them (keeps your local copy), and ignores the pattern. Loud about the one that matters: an untrack doesn't un-leak a pushed secret — rotate it. |

### ⚡ Token-efficient dev skills
| Skill | What it does | Why it's lean |
|-------|-------------|-------------|
| [`/lazy-commit`](skills/lazy-commit/) | Read diff → generate commit message → commit. One shot, zero chat. | No back-and-forth |
| [`/commit-split`](skills/commit-split/) | One messy working tree → several clean, one-concern commits. Groups hunks by intent (feature/fix/refactor/formatting) and stages each with `git add -p`. Stashes a backup first, never loses a change. | Untangle once, review forever |
| [`/squash`](skills/squash/) | The inverse of `/commit-split`: many junk commits (`wip`, `oops`, `fix test`) → one clean commit (or a few logical ones) before merge. Non-interactive `git reset --soft` to the merge-base — no `rebase -i`, zero conflicts. Backs up the branch, proves the tree is unchanged, pushes with `--force-with-lease`. | Collapse in one step, no rebase marathon |
| [`/changelog`](skills/changelog/) | Git history between two refs → grouped, reader-friendly release notes. Git does the work; AI just phrases it. | Pure CLI, AI only groups |
| [`/standup`](skills/standup/) | Your daily standup, rebuilt from the commits you actually made. Yesterday / Today / Blockers — grounded in git, never invented. | Git remembers; AI only phrases |
| [`/tldr-error`](skills/tldr-error/) | Paste error → 2-line diagnosis + fix. No lecture. | ~90% shorter responses |
| [`/triage`](skills/triage/) | A red CI job and a 2,000-line log → the one real failure (not the cascade), ranked causes, and the single command to run first. Reads to the root cause; quotes the log, never guesses. The step up from `/tldr-error` when the error is buried in noise. | Find line one, not line one hundred |
| [`/scaffold`](skills/scaffold/) | One-liner → production boilerplate. Detects your stack automatically. | Replaces 10 min of typing |
| [`/quick-test`](skills/quick-test/) | Point at a function → get a complete test file. No narration. | Tests are 80% boilerplate |
| [`/nit`](skills/nit/) | Self-review your own diff before the reviewer does. Flags the embarrassing stuff — a leftover `console.log`, a `describe.only` that quietly disables CI, commented-out code, a survived merge marker, a committed `.env`. Added lines only; flags, never fixes. Security is `/secure-diff`'s job. | One glance saves a review round |
| [`/pre-push`](skills/pre-push/) | "Ready to push?" → one go/no-go gate. Runs `/nit`, then `/secure-diff`, then your tests — fail-fast, worst-first, one verdict. A `.only` beats a green suite, so it blocks even when tests pass. Checks; doesn't commit. | The pre-flight checklist before `git push` |
| [`/pr-desc`](skills/pr-desc/) | Branch diff → PR title + description. Creates the PR if you want. | Never write a PR desc again |
| [`/unfuck`](skills/unfuck/) | Something broke. Reads errors + recent changes → finds root cause → fixes it. | The "just fix it" button |
| [`/rename-symbol`](skills/rename-symbol/) | Rename a variable/function across all files. Scope-aware, smarter than `sed`. | One command vs. manual find-replace |
| [`/orient`](skills/orient/) | Just cloned an unfamiliar repo? One-pass map — what it is, how to run it, how to test it, where the code lives, what's active. Reads manifests + CI (not the stale README); every command traces to a real script. `/explain` zoomed out. | One screen, then you're working — not spelunking |
| [`/explain`](skills/explain/) | Point at a function, file, regex, or gnarly one-liner → plain-English explanation, top-down. Reads the real code; never guesses from the name. | Purpose first, detail only as deep as it needs |
| [`/why`](skills/why/) | "Why is this line here? Safe to delete?" → chains `git blame` → the commit → the PR/issue to recover the original intent. Reads history, not the code; says so when the trail goes cold. | The reason lives in git, not the file |
| [`/todo`](skills/todo/) | Whole-repo sweep for `TODO`/`FIXME`/`HACK`/`XXX`/`BUG` markers → ranked, actionable worklist. Ages each with `git blame` (how old, whose, which commit) so a note-to-self from last week reads differently than a FIXME rotting since 2019. Inventories; doesn't auto-fix. `/nit` is your diff; this is the whole tree. | One ranked worklist, not a `grep` wall |
| [`/bisect`](skills/bisect/) | "When did this break?" → drives `git bisect` to the exact culprit commit. Automated with a test command, guided without one. | `log2(N)` steps, not N diffs |
| [`/oops`](skills/oops/) | Botched a git command? Reads `git reflog` to find your last-good state and walks back to it safely — bad reset, wrong-branch commit, deleted branch, mangled rebase, accidental `--amend`. | Reflog is a fact, not a guess |
| [`/conflict`](skills/conflict/) | Merge/rebase stopped with `CONFLICT`? Reads both sides' history per hunk, keeps both changes when they're compatible, asks before dropping a fix, sweeps leftover markers, and verifies before `--continue`. Knows ours/theirs flips in a rebase. | Resolve on merits, never coin-flip |
| [`/branch-cleanup`](skills/branch-cleanup/) | Forty local branches, six that matter? Fetch-prunes, then deletes only the provably-done ones — merged into trunk, plus the `[gone]` branches whose squash-merged PR deleted its remote (the case `--merged` can't see). Safe by default (`-d`, never `-D`); current and default branch always protected; unmerged work left alone. | Git already knows which are done — this reads it |
| [`/worktree`](skills/worktree/) | Bug on `main` while you're mid-feature? Instead of stashing or a throwaway commit, adds a linked `git worktree` — a second working directory on the *same* repo with another branch checked out, in its own sibling folder. Your work-in-progress sits untouched while you fix, review, or run two branches side by side. Branches the hotfix from the base (not your messy HEAD), removes cleanly with the porcelain (never `rm -rf`). | A second branch live, zero stashing, one repo |

### 🔬 Research & writing
| Skill | What it does |
|-------|-------------|
| [`/paper-review`](skills/paper-review/) | Senior IEEE-level peer reviewer. Factual integrity audits, originality checks, calibrated scoring. NeurIPS, ICML, IEEE S&P, USENIX Security. |
| [`/pradyoth-writing`](skills/pradyoth-writing/) | Ghostwrite in a sharp practitioner voice — LinkedIn, blogs, talk abstracts, threads. No corporate jargon, strong hooks. |

### 🎯 Strategy & mindset
| Skill | What it does |
|-------|-------------|
| [`/harvey-specter`](skills/harvey-specter/) | Think like the best closer in town. Predatory strategic thinking for negotiations, career moves, salary talks, power plays. Board reads, leverage maps, exact scripts. |

## Recipes

Step-by-step workflows that chain skills (or run bare with plain shell).

| Recipe | What it does |
|--------|-------------|
| [Pre-push safety net](recipes/pre-push-safety-net.md) | A two-minute ritual before every push: scan for secrets, run tests, audit new deps, write the PR. Catches the embarrassing stuff before it leaves your laptop. |
| [Hunt a regression](recipes/hunt-a-regression.md) | It worked last week, now it doesn't. Bisect to the exact culprit commit, understand it, fix the root cause, add the test, ship the PR. |

## Prompt templates

Battle-tested prompts for common dev tasks. Copy, fill the blanks, paste.

| Template | What it does |
|----------|-------------|
| [Rubber-duck debug](prompts/rubber-duck-debug.md) | Forces the model to find the *cause* before writing a fix — so you get a root cause, not a band-aid that lets the bug come back. |

## Repo layout

- **`/skills`** — custom Claude Code slash commands and reusable skills
- **`/recipes`** — step-by-step workflows that chain skills together
- **`/prompts`** — battle-tested prompt templates for common dev tasks
- **`/scripts`** — standalone scripts that pair well with Claude Code

## Philosophy

- If it takes more than 2 minutes manually, automate it
- If Claude can do it, write a skill for it
- If it's clever and saves time, it belongs here
- No over-engineering — jugaad is about pragmatic solutions
- If AI spends more tokens on an easy task than it's worth, skip the AI — saving tokens is also jugaad

## Contributing

PRs welcome. Keep it practical.

A good skill: does **one** thing, in **one** shot, with **clear triggers** in its `description` so Claude knows when to fire it. Look at any existing `SKILL.md` for the shape — YAML frontmatter (`name` + `description`), then numbered steps, then hard rules. If it needs a manual or a config file, it's too big.

Open an issue if you've got an idea but not the time. Half the skills here started as "ugh, I do this every day."

## License

MIT — do whatever you want with these.

---

<sub>Maintained with [Claude Code](https://claude.com/claude-code) — part of an [autonomous agent experiment](https://github.com/ppradyoth/social-experiment-with-agents) where AI agents keep these repos alive with zero human review for 23 days.</sub>
