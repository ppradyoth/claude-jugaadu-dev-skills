---
name: quick-test
description: >
  Generate a test file for a function, class, or module. No narration, just the test.
  Use when the user says "test this", "write tests for", "quick test", "add tests",
  or invokes /quick-test. Also trigger when the user points at a file and says "cover this".
---

# Quick Test

Point at code → get tests. No narration. No "here's what I'm testing". Just the test file.

## Steps

1. Read the target file/function
2. Detect the test framework from the project:
   - Python: check for pytest, unittest in existing tests or pyproject.toml
   - JS/TS: check for jest, vitest, mocha in package.json
   - Go: standard testing package
   - Rust: built-in #[test]
3. Find existing test files to match the project's test patterns:
   - File naming: `test_*.py`, `*.test.ts`, `*_test.go`, etc.
   - Directory: `tests/`, `__tests__/`, same dir, etc.
   - Import style, assertion style, fixture patterns
4. Generate tests covering:
   - Happy path (the obvious correct case)
   - Edge cases (empty input, None/null, boundary values)
   - Error cases (invalid input, expected exceptions)
5. Write the test file to the correct location
6. Run the tests. Fix if they fail. Output pass/fail summary.

## Rules

- NEVER explain what each test does — the test name should be self-documenting
- NEVER generate tests for private/internal methods unless asked
- NEVER mock things that are fast and deterministic (pure functions, utils)
- DO mock external services, databases, network calls
- Test BEHAVIOR, not implementation. Don't assert on internal state.
- One assertion per test (when reasonable). Multiple related assertions are fine.
- Test names should read like sentences: `test_returns_empty_list_when_no_items_match`
- If the function is trivial (getter, simple math), generate fewer tests — don't waste tokens on obvious stuff
- If the function is complex (multiple branches, state), be thorough
- Use parameterized tests for functions with many input/output pairs
- Always use the project's existing assertion style (assert vs expect vs should)

## Test Quality Checklist (internal — don't output this)

- [ ] Tests fail when the code is wrong (test the tests mentally)
- [ ] Tests don't depend on each other
- [ ] Tests don't depend on execution order
- [ ] No sleep() or time-dependent assertions
- [ ] No hardcoded file paths
- [ ] Tests clean up after themselves
