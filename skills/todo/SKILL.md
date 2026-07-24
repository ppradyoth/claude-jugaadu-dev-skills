---
name: todo
description: >
  Sweep the whole codebase for the TODO/FIXME/HACK/XXX/BUG markers everyone leaves and
  nobody revisits, then turn the pile into a ranked, actionable list — each one aged with
  git blame (how old, whose, which commit) so you can tell a note-to-self left last week
  from a landmine that's been rotting for two years. Use when the user says "find the
  TODOs", "what's left to do", "todo sweep", "any FIXMEs", "tech debt inventory", "what
  did we leave unfinished", or invokes /todo. Read-only inventory + triage — it does NOT
  fix the TODOs (that's the follow-up task) and it is NOT a diff self-review (that's /nit).
---

# Todo

Every codebase is quietly narrating its own regrets. `// TODO: handle the empty case`.
`# FIXME: this races under load`. `// HACK: hard-coded until the API ships`. They get typed
in a hurry, they never get grepped again, and six months later nobody remembers which ones
are harmless reminders and which one is the reason production falls over on Black Friday.

This skill does two things a raw `grep -r TODO` can't: it **dedupes and ranks** the markers
by how much they should scare you, and it **ages each one with git** — how long it's been
there, who wrote it, in what commit — because *a FIXME's danger is mostly its age.* A TODO
added yesterday is a plan. The same TODO untouched for three years is a lie the code has
been telling every reader since.

**The core move: turn scattered in-code markers into one ranked worklist, aged by git, so
you fix the landmines and ignore the sticky notes.**

## Scope

- **In scope:** finding every marker comment across the tree, classifying it, de-duplicating,
  aging it via `git blame`, and ranking it into a short worklist with a suggested next action
  (fix now / file an issue / delete the stale note / leave it).
- **Out of scope, on purpose:**
  - **Fixing the TODOs.** This produces the *list*. If the user says "now fix #1," that's the
    normal edit flow (or `/unfuck` if it's a live bug) — don't start editing during the sweep.
  - **Reviewing a diff for leftovers before push** → that's **`/nit`** (added lines only, one
    diff). `/todo` inventories the *whole repo's* existing markers, not your pending change.
  - **Recovering why one specific line exists** → that's **`/why`** (`git blame` → PR/issue for
    a single line). `/todo` uses blame only to *age* markers in bulk, not to reconstruct intent.

## Steps

1. **Find the markers across the tree, respecting `.gitignore`.** Use `git grep` so vendored
   and build dirs are excluded for free, and capture file + line:
   ```bash
   git grep -n -E -i '\b(TODO|FIXME|HACK|XXX|BUG|WIP|REFACTOR|OPTIMIZE|DEPRECATED)\b' \
     -- ':!*.lock' ':!*.min.*' ':!dist/' ':!build/' ':!vendor/' ':!node_modules/'
   ```
   If not a git repo, fall back to `rg -n -i` with the same alternation and `--glob` excludes.
   Match the token as a **word** (`\b`) so you don't flag `mastodon` or `fixmeup()`.

2. **Classify by marker, because the word tells you the severity.** Bucket every hit:
   - **BUG / FIXME** — an admitted defect. Highest priority; the author *knew* it was broken.
   - **HACK / XXX** — a known-fragile workaround. Breaks when an assumption shifts.
   - **TODO / WIP** — unfinished intent. Ranges from trivial to load-bearing.
   - **REFACTOR / OPTIMIZE / DEPRECATED** — quality/health debt, rarely urgent, easy to defer.

3. **Age each one with git — this is the step that makes the list useful.** For each marker,
   get when it was introduced and by whom:
   ```bash
   git blame -L <line>,<line> --porcelain -- <file>   # author + author-time + commit sha
   ```
   Batch it; don't blame the whole file. Convert author-time to an age. **Age reframes
   everything:** a `FIXME` from this sprint is a plan in motion; a `FIXME` from 2019 that
   everyone's editing *around* is either dead code or a normalized risk nobody owns.

4. **De-duplicate and cluster.** The same TODO copy-pasted across ten files is *one* problem,
   not ten. Group identical/near-identical text, and group by directory so "auth/ is full of
   FIXMEs" surfaces as a theme, not ten unrelated lines.

5. **Rank into a worklist.** Order by: marker severity (BUG/FIXME > HACK/XXX > TODO > health),
   then age (older = more suspicious for HACK/FIXME), then blast radius (in a hot path / core
   module beats a test fixture or an example). Read the surrounding line — a `TODO: also handle
   IPv6` in the request router outranks a `TODO: nicer copy` in a README.

6. **Suggest one next action per top item.** Not a fix — a *disposition*: fix now, file a
   tracked issue and reference it, or delete the note because the thing it describes already
   happened. Offer to open issues if there's a tracker.

## Output format

```
📋 /todo — <N> markers across <F> files  (BUG 3 · FIXME 8 · HACK 5 · TODO 21 · other 6)

🔴 Fix or file now
- src/auth/session.ts:88 — FIXME: token refresh races under load
  ↳ 2y3m old · alice@ · a1b2c3d — untouched, in a hot path. Landmine, not a note.
- src/pay/charge.go:142 — BUG: rounds down on split payments
  ↳ 5m old · bob@ · e4f5a6b — recent, admitted defect in money code. Fix or issue today.

🟠 Fragile workarounds (revisit)
- api/client.py:30 — HACK: hard-coded region until multi-region API ships
  ↳ 1y1m old · carol@ — has the "temporary until X" shipped yet? If yes, delete the hack.

🟡 Unfinished intent (batch when you touch the area)
- 6 TODOs clustered in src/import/  (oldest 1y8m) — treat as one "finish the importer" task.
- 21 TODOs total; 12 are >1y old — stale-note candidates, not active work.

🗑️  Likely dead
- utils/legacy.js:12 — TODO: remove after v2 migration
  ↳ 3y old · the v2 migration shipped in 2024. This note (and maybe the code) can go.

Next: want me to file issues for the 🔴 items, or start on one? (I inventory; I don't auto-fix.)
```

If the tree is clean:
```
📋 /todo — no TODO/FIXME/HACK/XXX/BUG markers found in tracked source. Clean (or well-hidden).
```

## Rules

1. **Inventory, don't fix.** The deliverable is the ranked list. Editing starts only when the
   user picks an item. A sweep that silently starts changing code is a surprise, not a service.
2. **Age is a first-class signal.** Never present markers as a flat list — always blame them.
   "How old and whose" is what separates a plan from a liability, and it's the whole reason to
   run this instead of `grep`.
3. **Severity follows the word.** `BUG`/`FIXME` are admissions of a defect and outrank a `TODO`
   wish. Don't bury a `FIXME` in money code under twenty cosmetic `TODO`s.
4. **Respect `.gitignore` and skip generated noise.** A TODO inside `node_modules`, a `.lock`,
   or a minified bundle is not the user's problem. `git grep` gets this right by default; keep
   the excludes when falling back to `rg`.
5. **Cluster before you count.** Ten copies of one marker is one worklist item. Report the
   theme ("auth/ is TODO-heavy"), not ten near-duplicate lines.
6. **Short list, not a data dump.** Surface the top handful that matter and *summarize the long
   tail* ("12 of 21 TODOs are >1y old"). A 200-line wall of every marker is the thing people
   already ignore — the value is the ranking.

## Gotchas

- **The word isn't always a marker.** `TODO` in a string literal, a variable named `todoList`,
  a Markdown checklist, or documentation *about* TODOs will match. `\b` word-matching helps;
  eyeball the line before ranking it high.
- **Old ≠ safe to delete.** A three-year-old `FIXME` can be load-bearing precisely *because*
  everyone learned to work around it. Age raises suspicion; it doesn't grant permission — read
  the code before suggesting deletion.
- **`git blame` follows the last edit, not the original.** A reformat, a mass-rename, or a
  `prettier` pass can reset the blame date and make an ancient TODO look new. If a file was
  recently reformatted, treat blame ages in it as a floor, and note it.
- **"Temporary until X" is the highest-value pattern.** Grep specifically for `until`,
  `for now`, `remove after`, `hard-coded`. Half of them describe a condition that already
  came true — the workaround is now just permanent, unreviewed behavior.
- **Don't file duplicate issues.** If a tracker reference already sits next to the marker
  (`// TODO(JIRA-123)`), it's tracked — surface it as "already ticketed," don't open another.
- **A repo with zero markers isn't necessarily clean.** Some teams ban TODO comments and use
  the issue tracker instead. Absence means "look in the tracker," not "no debt."
