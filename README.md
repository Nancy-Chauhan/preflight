# Preflight

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)]()
[![Claude Code](https://img.shields.io/badge/Claude%20Code-2.0+-purple.svg)]()
[![Skills](https://img.shields.io/badge/skills-1-blue.svg)]()

**Know exactly where your system breaks before you ship.**

Preflight is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that reads your codebase, traces every service call, and simulates what happens when real users hit your system. It finds the bottlenecks, cost cliffs, rate limits, and breaking points you'd otherwise discover in production.

```bash
git clone https://github.com/Nancy-Chauhan/preflight.git
cd your-project && claude --plugin-dir ../preflight
/preflight 500   # simulate at 500 users
```

---

## Table of Contents

- [What it finds](#what-it-finds)
- [Quick start](#quick-start)
- [Usage](#usage)
- [Export](#export)
- [What it covers](#what-it-covers)
- [How it works](#how-it-works)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

## What It Finds

- "Your Stripe webhook handler isn't idempotent -- a retry will charge the customer twice"
- "Your image upload accepts files up to 50MB but Cloudflare Workers has a 25MB request body limit"
- "At 500 orders/day you'll make 1,500 Stripe API calls -- that hits the 100/sec rate limit during flash sales"
- "Your inventory check and payment are separate calls -- two users can buy the last item at the same time"
- "Your Vercel Hobby plan has a 10s function timeout but your PDF invoice generation takes 40 seconds"
- "At 500 users you'll send 3,300 emails/month -- Resend free tier caps at 3,000"
- "If you change your EC2 instance, also update: Cloudflare DNS, Google OAuth callback, webhook URLs"

## Quick Start

```bash
# 1. Clone the plugin
git clone https://github.com/Nancy-Chauhan/preflight.git

# 2. Open your project in Claude Code with the plugin
cd your-project
claude --plugin-dir ../preflight

# 3. Run a preflight check for 500 users
/preflight 500
```

## Usage

```bash
/preflight               # Full report at 100 / 1K / 10K users
/preflight 500           # Simulate at a specific user count
/preflight map           # Infrastructure dependency map only
```

## Export

```bash
/preflight 500 --export md     # Markdown
/preflight 500 --export html   # Styled HTML with print layout
/preflight 500 --export pdf    # PDF via headless Chrome
/preflight 500 --export all    # All formats
/preflight map --export pdf    # Map only as PDF
```

| Format | Output file | Description |
|--------|-------------|-------------|
| `md` | `./preflight-report.md` | Shareable Markdown. Feed to LLMs for follow-up fixes. |
| `html` | `./preflight-report.html` | Self-contained HTML with color-coded severity badges. |
| `pdf` | `./preflight-report.pdf` | Generated from HTML. Falls back to Chromium/wkhtmltopdf/weasyprint. |
| `all` | All three files | Generates MD, HTML, and PDF in sequence. |

The `--export` flag is additive -- the inline report always displays in chat.

## What It Covers

### Services

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

### Capabilities

- Traces every API route, cron job, and webhook to the services it calls
- Detects multipliers ("for each user, call LLM once and send one email")
- Finds hard caps, concurrency limits, and timeout values in code
- Simulates daily volume: LLM calls, emails, DB queries at your target user count
- Checks if cron jobs can process all users within their time window
- Calculates when you hit rate limits and tier thresholds
- Projects data growth (DB size, storage, indexes)
- Maps environment variable and domain/URL dependency chains
- Flags cascading failures and multi-instance issues (in-memory state at scale)

## How It Works

- **Read-only analysis** -- reads your code with Claude's built-in tools. No dependencies installed, no code executed, nothing modified.
- **Detection + live pricing** -- ships with pricing data for 40+ services in `services.md`, and verifies current pricing via web search.
- **Structured skill** -- `SKILL.md` (analysis instructions), `services.md` (detection patterns + pricing), `examples.md` (output calibration).

## FAQ

**What do I need to run Preflight?**
Claude Code 2.0 or later. No other dependencies.

**Does it modify my code?**
No. Preflight is completely read-only. It doesn't install anything, run your code, or write to any files (unless you use `--export`).

**How does it know service pricing?**
It ships with pricing data for 40+ services and uses web search to verify current rates at analysis time.

**Can I add a service that isn't supported?**
Yes. Add an entry to `services.md` with package names, env variable patterns, pricing tiers, rate limits, and known gotchas. See [CONTRIBUTING.md](CONTRIBUTING.md).

**What languages and frameworks work?**
Any codebase works. Optimized detection for Next.js, Express, Fastify, Remix, Nuxt, Django, FastAPI, Flask, Rails, Gin, Echo, Actix, and Axum.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines. The easiest way to contribute is adding new services to `services.md`.

## License

MIT
