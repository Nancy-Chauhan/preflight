# Preflight - Claude Code Plugin

This is a Claude Code plugin that provides the `/preflight` skill for analyzing codebases.

## Project Structure

- `.claude-plugin/plugin.json` - Plugin manifest
- `skills/preflight/SKILL.md` - Core skill instructions (the main product)
- `skills/preflight/services.md` - Service detection database with pricing data
- `skills/preflight/examples.md` - Example output reports

## Key Principles

- The SKILL.md IS the product. It instructs Claude how to analyze any codebase.
- services.md is a reference database. The skill also uses WebSearch for current pricing.
- Every number in the output must come from the actual code, not generic benchmarks.
- The skill is read-only -- it never modifies the user's codebase.

## When Editing

- Keep SKILL.md under 500 lines
- When adding services to services.md, include: packages, env vars, ALL tier pricing, rate limits, gotchas
- Update examples.md when changing output format
- Update README.md supported services table when adding new services
