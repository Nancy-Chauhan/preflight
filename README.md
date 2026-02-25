# Preflight

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)]()
[![Claude Code](https://img.shields.io/badge/Claude%20Code-2.0+-purple.svg)]()
[![Skills](https://img.shields.io/badge/skills-1-blue.svg)]()

**Know exactly where your system breaks before you ship.**

Preflight is a Claude Code plugin that scans your codebase, traces every service call, and simulates what happens when real users hit your system. It finds the bottlenecks, cost cliffs, rate limits, and breaking points you'd otherwise discover in production.

## What It Does

Most cost calculators just list services and their pricing pages. Preflight actually **reads your code** and tells you:

- "Your newsletter cron has a `LIMIT 100` -- at 150 users, 50 people silently miss their newsletter"
- "Your RSS ingest needs 285 LLM calls but Gemini free tier is 15 RPM -- that's 19 minutes minimum"
- "Your Vercel Hobby plan has a 10s function timeout but your batch job takes 90 seconds"
- "At 500 users you'll send 3,300 emails/month -- Resend free tier caps at 3,000"
- "If you change your EC2 instance, also update: Cloudflare DNS, Google OAuth callback, webhook URLs"

## Example Output

```
# Preflight Report: MyApp
## Target: 500 monthly active users

## Bottlenecks & Breaking Points

[FAIL] Newsletter cron hard cap at 100 users
  File: src/lib/queries.ts:42
  Issue: LIMIT 100 in get_users_due_for_newsletter(). At 500 users,
         ~125 are due per hour. 25 users silently skipped.
  Fix: Implement pagination or loop until all users processed.

[FAIL] Vercel Hobby 10s function timeout
  File: src/app/api/newsletters/route.ts (maxDuration = 300)
  Issue: maxDuration=300 requires Vercel Pro. On Hobby, killed at 10s.
  Fix: Upgrade to Vercel Pro ($20/mo).

[WARN] Resend daily email limit
  File: src/lib/email/client.ts
  Issue: Free tier = 100 emails/day. You need 110+/day at 500 users.
  Fix: Upgrade to Resend Pro ($20/mo).

## Monthly Cost Summary

| Service     | 100 users | 500 users | 5,000 users |
|-------------|-----------|-----------|-------------|
| Resend      | $0        | $20       | $80         |
| Supabase    | $0        | $0        | $25         |
| Gemini      | $0        | $0        | $20         |
| Vercel      | $0        | $20       | $20         |
| **TOTAL**   | **$0**    | **$40**   | **$145**    |
```

## Install

```bash
claude /plugin install chauhan/preflight
```

Or manually:
```bash
git clone https://github.com/chauhan/preflight.git
claude --plugin-dir ./preflight
```

## Usage

### Full report (simulate at 100 / 1K / 10K users)
```
/preflight
```

### Simulate at a specific user count
```
/preflight 500
```

### Infrastructure dependency map only
```
/preflight map
```

### Export report to file
```
/preflight 500 --export md          # Save as Markdown
/preflight 500 --export html        # Save as HTML
/preflight 500 --export pdf         # Save as PDF
/preflight 500 --export all         # Save all formats
/preflight map --export pdf         # Map only, exported as PDF
```

Supported formats:
| Format | File | Notes |
|--------|------|-------|
| `md` | `./preflight-report.md` | Same content as inline report. Share with LLMs for follow-up fixes. |
| `html` | `./preflight-report.html` | Self-contained HTML with styled tables, color-coded severity badges, and A4 print layout. |
| `pdf` | `./preflight-report.pdf` | Generated from HTML via Chrome headless. Falls back to Chromium, wkhtmltopdf, or weasyprint. |
| `all` | All three files | Generates MD, HTML, and PDF in sequence. |

The `--export` flag is additive -- the inline report always displays in chat. Exported files are saved to the project root for easy discovery.

## What It Analyzes

### Service Detection
Scans your `package.json`, env files, config files, and imports to detect:

| Category | Services |
|----------|----------|
| AI/LLM | OpenAI, Anthropic, Gemini, Groq, Mistral, Replicate |
| Database | Supabase, Firebase, Neon, PlanetScale, MongoDB Atlas, Turso |
| Email | Resend, SendGrid, Postmark, AWS SES, Mailgun |
| Hosting | Vercel, Netlify, Fly.io, Railway, Render, AWS EC2, Cloudflare Workers |
| Auth | Auth0, Clerk, Kinde, NextAuth |
| Payments | Stripe, Paddle, LemonSqueezy |
| Storage | S3, Cloudflare R2, Cloudinary, Uploadthing |
| Analytics | PostHog, Mixpanel, Sentry, Datadog |
| Cache/Queue | Upstash Redis, Inngest, Trigger.dev, BullMQ |
| Messaging | Twilio, Svix |

### Flow Tracing
Traces how services are actually used in your code:
- Maps every API route, cron job, and webhook to the services it calls
- Detects multipliers ("for each user, call LLM once and send one email")
- Finds hard caps, concurrency limits, and timeout values in your code

### Operational Simulation
Simulates a real day at your target user count:
- How many LLM calls, emails, DB queries happen daily
- Whether your cron jobs can process all users within their time window
- When you'll hit rate limits and tier thresholds
- Data growth projections (DB size, storage, indexes)

### Bottleneck Detection
Finds breaking points with specific file references:
- Hard caps in your code that silently skip users
- Rate limit math (can your batch complete within the limit?)
- Timing conflicts (processing time vs function timeout)
- Cascading failures (what breaks when one service fails?)
- Multi-instance issues (in-memory state that won't work at scale)

### Infrastructure Dependency Map
Maps how your services connect:
- Environment variable dependency graph
- Domain/URL chain (DNS -> hosting -> OAuth -> webhooks)
- "If you change X, also update Y" table
- Migration pitfall flags

## Supported Frameworks

Works with any codebase. Optimized detection for:
- **JavaScript/TypeScript**: Next.js, Express, Fastify, Remix, Nuxt
- **Python**: Django, FastAPI, Flask
- **Go**: Standard library, Gin, Echo
- **Ruby**: Rails
- **Rust**: Actix, Axum

## How It Works

Preflight is a Claude Code skill -- a structured prompt that instructs Claude to analyze your codebase using its built-in tools (file reading, code search, web search). It doesn't install any dependencies or run any code in your project. It's read-only.

The skill includes:
- `SKILL.md` -- Step-by-step analysis instructions
- `services.md` -- Detection patterns and pricing data for 40+ services
- `examples.md` -- Example outputs for calibrating detail level

Pricing data in `services.md` is a reference starting point. The skill also uses web search to verify current pricing, so it stays up to date even as services change their plans.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

The easiest way to contribute is adding new services to `services.md`. Each service entry needs:
- Package names for detection
- Environment variable patterns
- All pricing tiers (not just free)
- Rate limits and quotas per tier
- Known gotchas

## License

MIT
