# Contributing to Preflight

## Adding a New Service

The easiest way to contribute is adding a new service to `skills/preflight/services.md`.

Each service entry needs:

```markdown
### Service Name
- **Packages**: `package-name` (npm, pip, etc.)
- **Env vars**: `SERVICE_API_KEY`, `SERVICE_URL`
- **Cost model**: Per-email / per-token / per-MAU / per-request / flat monthly
- **Tiers**:
  - Free: [limits]
  - Starter ($X/mo): [limits]
  - Pro ($X/mo): [limits]
- **Rate limits**: [per tier if different]
- **Gotchas**: [things developers commonly miss]
```

Requirements:
- Include ALL pricing tiers, not just free
- Include rate limits and quotas per tier
- Include at least one "gotcha" -- something developers commonly miss
- Use current pricing (check the service's pricing page)

## Improving the Skill

When editing `skills/preflight/SKILL.md`:
- Keep it under 500 lines
- Every instruction should produce concrete, code-specific output
- Avoid generic advice -- the skill should reference actual file paths and line numbers
- Test changes by running `/preflight` on a real project

## Improving Examples

When editing `skills/preflight/examples.md`:
- Examples should demonstrate realistic scenarios
- Include specific file references and line numbers
- Show the full range of output sections
- Include both PASS and FAIL findings

## Pull Request Process

1. Fork the repo
2. Create a branch (`add-service-name` or `improve-flow-tracing`)
3. Make your changes
4. Test by running the skill on a project
5. Submit a PR with a clear description of what changed and why
