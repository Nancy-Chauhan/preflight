# Example Preflight Reports

Use these examples to calibrate the level of detail, tone, and format of your output.

---

## Example 1: Small SaaS at 500 Users

```markdown
# Preflight Report: SkillFeed
## Target: 500 monthly active users

---

## Stack Detected

| Service | Category | Evidence | Current Tier | Files |
|---------|----------|----------|--------------|-------|
| Supabase | Database + Auth | @supabase/supabase-js, @supabase/ssr | Free | 18 |
| Resend | Email | resend | Free | 3 |
| Anthropic Claude | LLM | @anthropic-ai/sdk | Pay-as-you-go | 1 |
| Google Gemini | LLM | @google/generative-ai | Free | 1 |
| AgentMail | Email Ingestion | agentmail | Unknown | 2 |
| Svix | Webhook Verification | svix | Consumer only ($0) | 1 |
| Vercel | Hosting | vercel.json | Hobby (Free) | - |

---

## System Simulation: A Day at 500 Users

### Fixed Operations (run regardless of user count)
Every hour, the ingest cron fires (`/api/cron/ingest`, maxDuration=300s):
- **RSS ingestion**: Processes 57 feeds in batches of 5 with a 50-second time budget. Up to 5 items per feed = max 285 LLM categorization calls at 1,024 max tokens each. At Gemini free tier (15 RPM), this takes a minimum of 19 minutes. But the time budget is 50 seconds, so only ~12 feeds can be processed per run. The remaining 45 feeds are deferred.
- **Queue processing**: Fetches up to 10 pending AgentMail ingestion jobs. Each triggers 1 LLM call at 4,096 max tokens.

### User-Triggered Operations
- **Signups** (estimating 5/day at 500 MAU):
  - 5 Supabase auth events (OAuth/magic link)
  - 5 LLM calls for profile parsing (4,096 max tokens each)
  - 5 Resend emails (waitlist confirmation)
  - 5 Resend emails (approval, assuming same-day approval)

### Scheduled Per-User Operations
- **Newsletter generation** (`/api/newsletters/generate-all`, hourly):
  - `get_users_due_for_newsletter(p_target_hour=8)` returns users where it's 8 AM local time
  - With 500 users across ~4 timezones, roughly 125 users due per target hour
  - BUT: the query has `LIMIT 100` -- only 100 users processed per cron run
  - At concurrency 5, each batch of 5 users takes ~3-5s (LLM call + MJML render + email send)
  - 100 users = 20 batches = 60-100 seconds. Within the 300s maxDuration.
  - Each user triggers: 1 Supabase RPC (match_articles), 1 LLM call (4,096 max tokens), 1 Resend email

### Daily Totals
| Resource | Daily Volume | Monthly Volume |
|----------|-------------|----------------|
| LLM calls (newsletter) | 100 | 3,000 |
| LLM calls (RSS ingest) | ~285 | ~8,550 |
| LLM calls (queue) | ~10 | ~300 |
| LLM calls (profile parse) | ~5 | ~150 |
| Resend emails (newsletter) | 100 | 3,000 |
| Resend emails (signup) | ~10 | ~300 |
| Supabase DB queries | ~500 | ~15,000 |
| Supabase auth events | ~5 | ~150 |

---

## Resource Usage & Cost Projection

### Resend
Current tier: Free (3,000 emails/month, 100 emails/day)
Projected usage: ~3,300 emails/month (3,000 newsletters + 300 signup emails)
Daily peak: 100 newsletters + 10 signup = 110 emails/day (EXCEEDS 100/day free limit)
**Free tier maxes out at: ~90 users (daily limit hit first)**
Next tier: Pro ($20/month) - 50,000 emails/month, no daily limit
Pro tier maxes out at: ~1,500 users
After that: Business ($80/month) - 200,000 emails/month

### Supabase
Current tier: Free (500 MB database, 50K MAU)
Projected DB growth: ~285 article rows/day x 365 = ~104K rows/year, estimated 50-100 MB
Projected MAU: 500 (well within 50K limit)
**Free tier sufficient until: ~50,000 MAU or 500 MB database**
Next tier: Pro ($25/month) - 8 GB database, 100K MAU

### Google Gemini (LLM - assuming primary provider)
Current tier: Free (15 RPM, 1,500 requests/day)
Projected daily requests: ~400 (285 RSS + 100 newsletter + 10 queue + 5 profile)
Projected RPM peak: During RSS ingest, could hit 15 RPM limit
**Free tier is feasible but rate-limited. RSS ingest will be throttled.**
Next tier: Pay-as-you-go with Gemini 2.0 Flash ($0.10/1M input, $0.40/1M output)
Estimated monthly cost at 500 users: ~$2-5 (depending on avg token usage)

### Anthropic Claude (if used instead of Gemini)
Projected usage: ~12,000 calls/month x avg 2K tokens = ~24M tokens/month
Estimated cost with Sonnet 4: ~$72/month input + ~$360/month output = ~$432/month
**Significant cost difference vs Gemini free tier**

### Vercel
Current tier: Hobby (100K function invocations, 10s timeout)
Projected invocations: Cron runs + API calls + page loads = ~50K/month
**CRITICAL: 10s function timeout on Hobby. Newsletter generation takes 60-100s.
This will NOT work on Hobby tier.**
Next tier: Pro ($20/month) - 300s timeout, 1M invocations

### Monthly Cost Summary

| Service | Free Tier | At 500 Users | At 5,000 Users |
|---------|-----------|-------------|----------------|
| Resend | $0 | $20/mo (Pro) | $80/mo (Business) |
| Supabase | $0 | $0 (free) | $25/mo (Pro) |
| Gemini | $0 | $0 (free, throttled) | ~$20/mo (paid) |
| Vercel | $0 | $20/mo (Pro required) | $20/mo |
| **TOTAL** | **$0** | **$40/mo** | **$145/mo** |

---

## Bottlenecks & Breaking Points

[FAIL] Newsletter cron hard cap at 100 users
  File: src/lib/supabase/queries.ts (get_users_due_for_newsletter RPC)
  Issue: LIMIT 100 in the database function. With 500 users across ~4 timezones,
         ~125 users are due at 8 AM. 25 users will NOT receive their newsletter.
  Impact: ~5% of users silently miss their daily newsletter. No error logged.
  Fix: Implement pagination -- run the cron multiple times or loop until all
       users are processed.

[FAIL] Vercel Hobby 10s function timeout
  File: src/app/api/newsletters/generate-all/route.ts (maxDuration = 300)
  Issue: maxDuration=300 requires Vercel Pro. On Hobby, the function will timeout
         at 10s, processing only ~3 users before being killed.
  Impact: Newsletter generation completely broken on Hobby tier.
  Fix: Upgrade to Vercel Pro ($20/mo) or move to self-hosted.

[WARN] Resend daily limit blocks newsletters
  File: src/lib/resend/client.ts
  Issue: Free tier allows only 100 emails/day. At 500 users, you need 110+/day.
  Impact: Some users won't receive newsletters on any given day.
  Fix: Upgrade to Resend Pro ($20/mo).

[WARN] RSS ingest exceeds time budget
  File: src/app/api/cron/ingest/route.ts
  Issue: 57 feeds at 15 RPM = 19 minutes minimum. Time budget is 50 seconds.
         Only ~12 feeds processed per hourly cron run.
  Impact: Many RSS feeds are stale for hours. New articles are delayed.
  Fix: Switch to Gemini paid tier (2,000 RPM) or process feeds across multiple
       cron runs with rotation.

[WARN] In-memory rate limiter breaks on multi-instance
  File: src/lib/utils/rate-limiter.ts
  Issue: Rate limiter uses an in-memory Map. On Vercel, each function invocation
         is a separate instance. The rate limiter has no effect.
  Impact: No actual rate limiting in production on Vercel.
  Fix: Use Upstash Redis-based rate limiter, or accept the risk if traffic is low.

[PASS] Supabase connection limits
  Free tier allows 60 connections. At concurrency 5 for newsletters + normal
  traffic, peak connections ~10-15. Well within limits.

[PASS] Database storage
  At 285 rows/day, database growth is ~50 MB/year. Free tier 500 MB is sufficient
  for 5+ years.

---

## Infrastructure Dependency Map

```
[Domain: skillfeed.dev]
  |
  +-- DNS (Cloudflare?) -> Vercel deployment URL
  |
  +-- Vercel hosting
  |     +-- Environment variables (18 vars across services)
  |     +-- Cron jobs (ingest: hourly, newsletters: hourly)
  |
  +-- Supabase (us-east-1)
  |     +-- Auth: Google OAuth callback -> skillfeed.dev/auth/callback
  |     +-- Auth: Site URL -> https://skillfeed.dev
  |     +-- Database: users, articles, newsletters_sent, waitlist, ingestion_jobs
  |
  +-- Resend
  |     +-- Verified domain: skillfeed.dev
  |     +-- FROM address: configured in EMAIL_FROM
  |
  +-- AgentMail
  |     +-- Webhook endpoint -> skillfeed.dev/api/webhooks/agentmail
  |     +-- Svix signature verification
  |
  +-- Google OAuth
        +-- Authorized redirect URI -> skillfeed.dev/auth/callback
```

### Change Impact Table

| If You Change... | Also Update... |
|------------------|----------------|
| Hosting (Vercel -> EC2) | DNS records, cron scheduler (need external like GitHub Actions), `after()` usage (won't work outside Vercel), env vars on new host, maxDuration settings |
| Domain name | DNS, Google OAuth redirect URI, Supabase Site URL + Redirect URLs, Resend verified domain, AgentMail webhook URL, any hardcoded URLs in code, HMAC signing keys if domain-dependent |
| Supabase project | All SUPABASE_* env vars, run migrations on new project, seed data, update Google OAuth callback in Supabase dashboard |
| LLM provider (Gemini -> Claude) | LLM_PROVIDER env var, ANTHROPIC_API_KEY, budget ~$400/mo vs $0 |

---

## Recommendations

### Before Launch (Fix These)
1. **Upgrade Vercel to Pro** ($20/mo) - newsletter generation is completely broken on Hobby
2. **Upgrade Resend to Pro** ($20/mo) - daily email limit will block newsletters
3. **Fix the 100-user newsletter cap** - implement pagination in the cron job

### Monitor Closely
4. RSS ingest throughput - feeds are being processed much slower than intended
5. In-memory rate limiter - provides no protection on Vercel
6. LLM token usage if switching from Gemini free to paid tier

### Can Wait (Only at 5,000+ Users)
7. Supabase upgrade to Pro
8. Newsletter concurrency tuning
9. Article table indexing strategy
```

---

## Example 2: Self-Hosted App at 5,000 Users

```markdown
# Preflight Report: InvoiceApp
## Target: 5,000 monthly active users

---

## Stack Detected

| Service | Category | Evidence | Current Tier | Files |
|---------|----------|----------|--------------|-------|
| Supabase | Database | @supabase/supabase-js, DATABASE_URL | Pro ($25/mo) | 24 |
| Stripe | Payments | stripe, @stripe/stripe-js | Standard | 8 |
| OpenAI | LLM | openai | Tier 2 | 4 |
| SendGrid | Email | @sendgrid/mail | Essentials ($19.95/mo) | 3 |
| Cloudflare | DNS + CDN | CF_ACCOUNT_ID, wrangler.toml | Pro ($20/mo) | - |
| AWS EC2 | Hosting | Dockerfile, docker-compose.yml | t3.medium ($30/mo) | - |
| Sentry | Error Tracking | @sentry/nextjs | Team ($26/mo) | 6 |
| Upstash Redis | Cache | @upstash/redis | Pay-as-you-go | 5 |

---

## System Simulation: A Day at 5,000 Users

### User-Triggered Operations
- **Invoice creation** (~200 invoices/day at 5K MAU):
  - 200 Supabase inserts (invoice + line items)
  - 200 OpenAI calls for "smart description" generation (GPT-4o mini, ~500 tokens each)
  - 200 PDF generations (server-side, CPU-intensive)
  - 200 SendGrid emails (invoice delivery)
  - 200 Stripe payment link creations

- **Payment processing** (~80 payments/day, 40% conversion):
  - 80 Stripe webhook events
  - 80 Supabase updates (invoice status)
  - 80 SendGrid emails (payment confirmation)
  - 80 OpenAI calls for receipt summary (GPT-4o mini)

- **Dashboard loads** (~2,000/day):
  - 2,000 Supabase queries (user's invoices, stats)
  - 2,000 Redis cache checks (dashboard data cached 5 min)
  - ~400 Redis cache misses -> Supabase aggregate queries

### Daily Totals
| Resource | Daily | Monthly |
|----------|-------|---------|
| SendGrid emails | 280 | 8,400 |
| OpenAI calls (GPT-4o mini) | 280 | 8,400 |
| Stripe transactions | 80 | 2,400 |
| Supabase queries | ~3,000 | ~90,000 |
| Redis commands | ~4,400 | ~132,000 |
| PDF generations | 200 | 6,000 |

---

## Resource Usage & Cost Projection

### Stripe
Current: Standard pricing (2.9% + $0.30)
Average invoice: $250
Monthly GMV: 2,400 payments x $250 = $600,000
Stripe fees: $600,000 x 2.9% + (2,400 x $0.30) = $17,400 + $720 = **$18,120/month**
Chargeback risk at 1%: 24 chargebacks x $15 = $360/month
**This is your largest cost by far.**

### OpenAI (GPT-4o mini)
Projected: 8,400 calls/month x ~500 tokens avg = ~4.2M tokens
Cost: 4.2M input x $0.15/1M + 4.2M output x $0.60/1M = **~$3.15/month**
Rate limit check: 280 calls/day = ~12/hour. Well within Tier 2 limits.

### SendGrid
Current tier: Essentials ($19.95/mo, 50K emails)
Projected: 8,400 emails/month
**Essentials tier is sufficient until ~15,000 users.**

### AWS EC2 (t3.medium)
Current: t3.medium ($30/mo) - 2 vCPU, 4 GB RAM
PDF generation is CPU-intensive. 200 PDFs/day at ~2s each = 400s/day of high CPU.
With t3 burstable credits: earns 24 credits/day, uses ~7 credits for PDF bursts.
**Sufficient, but monitor CPU credit balance. At 10K+ users, consider t3.large.**
Additional costs: EBS ($10/mo for 50 GB), data transfer (~$5/mo for 50 GB egress)

### Upstash Redis
Projected: 132K commands/month
On pay-as-you-go: 132K x $0.2/100K = **$0.26/month**
**Negligible cost.**

### Monthly Cost Summary

| Service | 1,000 Users | 5,000 Users | 10,000 Users |
|---------|-------------|-------------|--------------|
| Stripe fees | $3,624 | $18,120 | $36,240 |
| Supabase | $25 | $25 | $25-50 |
| SendGrid | $19.95 | $19.95 | $89.95 |
| EC2 + EBS | $45 | $45 | $75 |
| OpenAI | $0.63 | $3.15 | $6.30 |
| Cloudflare | $20 | $20 | $20 |
| Sentry | $26 | $26 | $26-80 |
| Redis | $0.05 | $0.26 | $0.52 |
| **TOTAL** | **$3,761** | **$18,259** | **$36,487** |
| **TOTAL (ex-Stripe)** | **$137** | **$139** | **$247** |

**Key insight**: Infrastructure costs are negligible ($139/mo). Stripe fees dominate at $18,120/mo. Consider Stripe volume discounts at this GMV.

---

## Bottlenecks & Breaking Points

[WARN] PDF generation CPU spikes
  File: src/lib/pdf/generator.ts
  Issue: Synchronous PDF rendering blocks the event loop for ~2s per PDF.
         At 200 PDFs/day arriving in bursts, CPU can spike to 100%.
  Impact: Other requests slow down during invoice creation peaks.
  Fix: Move PDF generation to a background queue (BullMQ). Process async.

[WARN] No connection pooling for Supabase
  File: src/lib/supabase/client.ts
  Issue: Creating new Supabase client per request. At peak load (~50 concurrent
         requests), could exhaust the 200-connection Pro tier limit.
  Impact: Database connection errors under load.
  Fix: Use Supabase connection pooler (PgBouncer) via the pooled connection string.

[WARN] Stripe webhook has no idempotency check
  File: src/app/api/webhooks/stripe/route.ts
  Issue: Stripe can retry webhooks up to 3 times. No idempotency key check.
         Could process the same payment twice.
  Impact: Duplicate invoice status updates, potential double email sends.
  Fix: Store processed event IDs and skip duplicates.

[PASS] SendGrid limits - well within Essentials tier at 5K users
[PASS] OpenAI rate limits - 12 RPM is far below Tier 2 limits
[PASS] Redis command limits - negligible usage

---

## Infrastructure Dependency Map

```
[Domain: invoiceapp.com]
  |
  +-- Cloudflare DNS -> A record -> EC2 elastic IP (54.x.x.x)
  |     +-- SSL: Cloudflare origin certificate on EC2
  |     +-- CDN: Static assets cached at edge
  |
  +-- EC2 (t3.medium, us-east-1)
  |     +-- Docker container: Next.js app
  |     +-- EBS volume: 50 GB (app + logs)
  |     +-- Security group: ports 80, 443, 22
  |
  +-- Supabase (us-east-1)
  |     +-- Auth redirect -> https://invoiceapp.com/auth/callback
  |     +-- Site URL -> https://invoiceapp.com
  |
  +-- Stripe
  |     +-- Webhook endpoint -> https://invoiceapp.com/api/webhooks/stripe
  |     +-- Success/cancel URLs -> https://invoiceapp.com/invoices/*
  |
  +-- SendGrid
        +-- Verified domain: invoiceapp.com
        +-- DKIM/SPF DNS records in Cloudflare
```

### Change Impact Table

| If You Change... | Also Update... |
|------------------|----------------|
| EC2 instance (new IP) | Cloudflare DNS A record, SSH keys, security groups, EBS reattach, env vars, SSL cert if not using Cloudflare proxy |
| Domain name | Cloudflare zone, Supabase Site URL + redirect URLs, Stripe webhook URL + success/cancel URLs, SendGrid verified domain + DNS records (DKIM, SPF, DMARC), any hardcoded URLs |
| Database (Supabase project) | DATABASE_URL, SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY, run migrations, re-configure auth providers in new project dashboard |
| Payment processor (Stripe -> Paddle) | Webhook handler rewrite, checkout flow rewrite, subscription management, remove Stripe SDK, add Paddle SDK, update env vars, tax handling changes |

---

## Recommendations

### Before Launch
1. **Add Stripe webhook idempotency** - risk of duplicate processing
2. **Move PDF generation to background queue** - will cause timeouts under load

### Monitor Closely
3. EC2 CPU credit balance during peak hours
4. Supabase connection count (consider pooler)
5. Stripe fees as GMV grows (negotiate volume pricing at $500K+/mo)

### Can Wait
6. SendGrid upgrade (only at ~15K users)
7. EC2 instance upgrade to t3.large (only at ~10K users)
```

---

## Example 3: Infrastructure Map Only (`/preflight map`)

```markdown
# Preflight Infrastructure Map: MyApp

## Services Detected

| Service | Category | Env Vars | Files |
|---------|----------|----------|-------|
| Vercel | Hosting | VERCEL_URL | - |
| Supabase | Database + Auth | 4 vars | 12 |
| Clerk | Auth | 3 vars | 6 |
| Resend | Email | 2 vars | 2 |
| Stripe | Payments | 3 vars | 5 |
| PostHog | Analytics | 2 vars | 3 |

## Environment Variable Map

| Variable | Service | Referenced In |
|----------|---------|---------------|
| NEXT_PUBLIC_SUPABASE_URL | Supabase | 8 files |
| NEXT_PUBLIC_SUPABASE_ANON_KEY | Supabase | 8 files |
| SUPABASE_SERVICE_ROLE_KEY | Supabase | 3 files |
| DATABASE_URL | Supabase (Postgres) | prisma/schema.prisma |
| NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY | Clerk | 4 files |
| CLERK_SECRET_KEY | Clerk | 2 files |
| CLERK_WEBHOOK_SECRET | Clerk | 1 file |
| RESEND_API_KEY | Resend | 1 file |
| EMAIL_FROM | Resend | 1 file |
| STRIPE_SECRET_KEY | Stripe | 3 files |
| STRIPE_PUBLISHABLE_KEY | Stripe | 2 files |
| STRIPE_WEBHOOK_SECRET | Stripe | 1 file |
| NEXT_PUBLIC_POSTHOG_KEY | PostHog | 2 files |
| POSTHOG_HOST | PostHog | 1 file |

## Dependency Graph

```
[Domain: myapp.com]
  |
  +-- Vercel (hosting)
  |     +-- All env vars configured in Vercel dashboard
  |     +-- Auto-deploy from GitHub main branch
  |
  +-- Supabase
  |     +-- Database: PostgreSQL (via Prisma ORM)
  |     +-- NOTE: Both Supabase Auth AND Clerk detected.
  |     |   Potential conflict or migration in progress?
  |     +-- Supabase site URL -> https://myapp.com
  |
  +-- Clerk
  |     +-- Webhook -> https://myapp.com/api/webhooks/clerk
  |     +-- Handles sign-in/sign-up UI
  |
  +-- Stripe
  |     +-- Webhook -> https://myapp.com/api/webhooks/stripe
  |     +-- Checkout success URL -> https://myapp.com/success
  |
  +-- Resend
  |     +-- Verified domain: myapp.com
  |
  +-- PostHog
        +-- Client-side tracking (NEXT_PUBLIC key)
        +-- Host: app.posthog.com (cloud)
```

## Change Impact Table

| If You Change... | Also Update... |
|------------------|----------------|
| Hosting (Vercel -> self-hosted) | Set up reverse proxy (nginx), configure all 14 env vars on new host, set up SSL (Let's Encrypt), set up CI/CD for deploys, DNS A record to new server IP, cron jobs (Vercel cron won't work) |
| Domain | Vercel domain settings, Supabase site URL, Clerk production domain + webhook URL, Stripe webhook URL + checkout URLs, Resend verified domain + DNS records, PostHog allowed origins, any hardcoded URLs |
| Auth (Clerk -> Supabase Auth) | Remove Clerk packages + env vars, rewrite auth middleware, migrate user data, update webhook handler, remove Clerk UI components, implement Supabase auth UI |
| Database (Supabase -> Neon) | Update DATABASE_URL, keep or replace Supabase for auth/storage, update Prisma connection, run migrations on new DB, seed data migration |

## Warnings

[WARN] Dual auth providers detected
  Both Supabase Auth (@supabase/ssr) and Clerk (@clerk/nextjs) are installed.
  This may indicate a migration in progress. Verify which is actively used
  and remove the unused one to avoid confusion and unnecessary costs.
```
