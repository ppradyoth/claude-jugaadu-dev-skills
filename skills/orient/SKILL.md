---
name: orient
description: >
  Dropped into an unfamiliar repo? Map it in one pass — what it is, how to run it,
  where the code lives, how to test it — so you can start working, not spelunking.
  Use when the user says "orient me", "what is this repo", "help me understand this
  codebase", "I just cloned this", "where do I start", "give me the lay of the land",
  "onboard me to this project", or invokes /orient. Reads the real files; never guesses
  from the repo name. One screen, not a tour.
---

# Orient

Someone just landed in a codebase they don't know — a new job, an open-source repo they want to contribute to, a service they got paged about at 2am. They don't want a wiki. They want the *one screen* that lets them start doing work: what this is, how to run it, where the important code lives, and how to prove a change works.

This reads the actual repo and produces that screen. Top-down. Facts from files, not vibes from the README's marketing.

## Steps

1. **Read the ground truth, in this order.** Don't guess from the repo name.
   - **Manifests** — `package.json`, `pyproject.toml` / `requirements.txt`, `go.mod`, `Cargo.toml`, `pom.xml`, `Gemfile`, `composer.json`. These name the language, the deps, and — critically — the **scripts** (`scripts` block, `Makefile`, `Taskfile`, `justfile`). The run/build/test commands are usually sitting right here.
   - **README / CONTRIBUTING / docs** — but treat them as *claims to verify*, not truth. READMEs rot. If the README says `npm start` but there's no `start` script, trust the manifest.
   - **Entry points** — `main`, `index`, `app`, `cmd/`, `src/main`, the `bin` field, a `Dockerfile`'s `CMD`/`ENTRYPOINT`, `docker-compose.yml` services. Where does execution actually begin?
   - **Config & env** — `.env.example`, `config/`, `settings`, CI files (`.github/workflows`, `.gitlab-ci.yml`). CI is the most honest doc in any repo: it's the exact commands that *must* pass.

2. **Map the layout, not every file.** Read the top-level directory tree one or two levels deep. Name the handful of directories that matter (where the source lives, where tests live, where config lives) and skip the noise (`node_modules`, `dist`, `vendor`, `.venv`).

3. **Find how it's tested.** Test framework (from the manifest + a glance at a test file), where tests live, and the exact command to run them. This is the single most valuable thing for a newcomer — it's how they'll know their change didn't break anything.

4. **Take the pulse (optional, cheap).** `git log --oneline -10` and the most-recently-touched files show what's *active* right now — where the current work is, which corners are dead. One command, high signal.

5. **Emit the orientation.** One screen, in this shape (drop any section the repo genuinely doesn't have — don't pad):

   ```
   WHAT      One line: what this project is and does. In the user's terms.
   STACK     Language(s), framework(s), notable deps. Version if it's pinned and matters.
   RUN       The exact command(s) to install and start it. Copy-paste ready.
   TEST      The exact command to run the tests. Framework + where they live.
   LAYOUT    3-6 dirs that matter, one line each. src → what, tests → what, etc.
   ENTRY     Where execution starts — the file to open first.
   ACTIVE    What's been worked on lately (from git). Where to look for live code.
   GOTCHAS   Anything surprising: monorepo, required env vars, a build step before run,
             a service dependency (DB/Redis), a non-obvious "you must do X first".
   ```

## Rules

- **Read, don't assume.** Every command you emit (`RUN`, `TEST`) must trace to a real script, Makefile target, or CI step you actually saw. If you're inferring a command that isn't written down anywhere, say so — `(inferred, not in any script)`.
- **The manifest beats the README.** When they disagree, trust the executable source (scripts/CI), and flag the stale doc as a gotcha.
- **Name files so they're clickable** — `src/server.ts:1`, `cmd/api/main.go`. The point is to send the reader somewhere, not describe somewhere.
- **One screen.** This is orientation, not documentation. If it takes more than a screen, you're explaining the whole codebase — that's not the job. Breadth over depth; the reader drills in themselves.
- **Don't invent structure.** If there are no tests, say "no tests found" — that itself is orientation (and a gotcha). Never fabricate a `TEST` command to look complete.
- **Skip generated/vendored dirs** in the layout — they're never where the reader needs to look first.
- **If the repo is trivial** (a single script, a config repo, a docs site), say that in a line and give the one thing that matters. Don't force the full template onto a 3-file repo.

## Gotchas

- **Monorepos.** If you see `packages/`, `apps/`, a workspaces field, or multiple manifests, the run/test commands are usually *per-package* — say which package, or that the reader must `cd` first. One global command is often wrong here.
- **The README lies more often than the code.** A `RUN` command copied from a stale README that no longer works is worse than saying "no documented run command found" — it burns the newcomer's first ten minutes. Verify against scripts/CI.
- **`.env.example` is a checklist, not decoration.** If it exists, the app almost certainly won't start without those vars set. Surface that in `GOTCHAS`, don't bury it.
- **Don't actually run install/build to "check"** unless the user asked you to get it running — orientation is read-only reconnaissance. Installing deps in someone's fresh clone is a side effect they didn't ask for. (If they *do* want it running, that's `/unfuck` territory when it breaks.)
- **This is `/explain` zoomed out.** `/explain` tells you what one function/file does; `/orient` tells you what the whole repo is and where to point `/explain` next. Use them in sequence: orient to find the entry point, explain to understand it.
