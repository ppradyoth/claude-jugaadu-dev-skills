---
name: scaffold
description: >
  Generate boilerplate code from a one-liner description. Use when the user says
  "scaffold", "generate a", "create a new", "boilerplate for", "stub out",
  "skeleton for", or invokes /scaffold. Covers API routes, components, CLI tools,
  test files, scripts, config files, Docker setups, and more.
---

# Scaffold

Generate production-ready boilerplate from a one-liner. No back-and-forth.

## Steps

1. Parse what the user wants (component, route, script, test, etc.)
2. Detect the project's stack from existing files:
   - Check `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, etc.
   - Match the existing code style (tabs vs spaces, quotes, semicolons)
   - Use the same import patterns the project already uses
3. Generate the file(s) and write them to disk
4. Output only the file path(s) created. No explanation.

## Rules

- NEVER ask "what framework are you using?" — detect it from the project
- NEVER generate placeholder comments like `// TODO: implement this`
- NEVER add dependencies the project doesn't already use without saying so
- Match the project's existing patterns EXACTLY (naming conventions, folder structure, export style)
- If the project uses TypeScript, generate TypeScript. If JS, generate JS. Never mix.
- Include only the imports actually needed
- No default exports unless the project uses them
- No prop-types if the project uses TypeScript

## Templates by Type

### API Route / Endpoint
- Include request validation
- Include error response pattern matching the project's existing routes
- Include the appropriate HTTP methods

### React/Vue/Svelte Component
- Match the project's component pattern (functional, class, composition API, etc.)
- Include props interface/type if TypeScript
- No unnecessary state or effects

### Test File
- Use the project's test framework (jest, vitest, pytest, go test)
- Include one passing test as a sanity check
- Structure: describe block with the function/component name

### CLI Script
- Include argument parsing (argparse, commander, clap — whatever the project uses)
- Include `--help` output
- Include error exit codes

### Python Script
- Include `if __name__ == "__main__":` guard
- Type hints if the project uses them

### Docker
- Multi-stage build if applicable
- `.dockerignore` alongside `Dockerfile`
- Use slim/alpine base images
