---
name: rename-symbol
description: >
  Rename a variable, function, class, or type across all files, respecting scope and
  language semantics. Use when the user says "rename X to Y", "refactor name",
  "rename symbol", "change function name", or invokes /rename-symbol. Smarter than
  sed, cheaper than an LSP.
---

# Rename Symbol

Rename a symbol across the codebase. Smarter than find-and-replace, cheaper than firing up an LSP.

## Steps

1. Identify what's being renamed: `old_name` → `new_name`
2. Find all occurrences with context:
   ```bash
   grep -rn --include='*.{ts,tsx,js,jsx,py,go,rs,java}' '\bold_name\b' .
   ```
3. Classify each occurrence:
   - **Definition**: where the symbol is declared (function def, class def, variable assignment)
   - **Usage**: where it's called/referenced
   - **String literal**: inside quotes — probably NOT a rename target
   - **Comment**: inside a comment — rename only if it references the symbol
   - **Import/Export**: module boundaries — always rename
   - **Test**: test files — always rename
   - **Config/JSON**: config files — rename if it's a key that maps to the code symbol

4. Apply the rename:
   - Use the Edit tool for each file (NOT sed — Edit is safer and reversible)
   - Rename definition first, then usages, then imports/exports
   - Handle destructured imports: `{ oldName }` → `{ newName }`
   - Handle re-exports: `export { oldName }` → `export { newName }`
   - Handle string references in tests: `describe('oldName'` → `describe('newName'`

5. Verify:
   - Run `grep -rn '\bold_name\b' .` to check for any missed occurrences
   - If the project has a build step, run it to catch compile errors
   - Report what was changed: `Renamed X occurrences across Y files`

## Rules

- NEVER rename occurrences inside string literals unless they're clearly referencing the code symbol (like test descriptions or error messages that name the function)
- NEVER rename partial matches — `userName` should not touch `userNameInput` when renaming `userName` to `displayName`. Use word boundaries.
- NEVER rename across package/module boundaries you don't control (node_modules, site-packages)
- If the symbol is exported from a library/package, warn that downstream consumers will break
- Handle case-sensitive renames: renaming `user` to `account` should also catch `User` → `Account`, `USER` → `ACCOUNT` if the user confirms
- For Python: handle `snake_case` and check `__init__.py` re-exports
- For JS/TS: handle both named and default exports, handle barrel files (index.ts re-exports)
- If >50 occurrences, show a preview of the first 10 and ask for confirmation before proceeding
- Show a final diff summary (files changed, occurrences replaced)
