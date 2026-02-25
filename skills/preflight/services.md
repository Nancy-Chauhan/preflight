# Service Detection Database

This file is a reference for the preflight skill. Use it to:
1. Match detected packages/env vars to services
2. Look up pricing across all tiers (not just free)
3. Find rate limits and quotas per tier
4. Identify known gotchas for each service

If a service is not listed here, use WebSearch to find current pricing and limits.

---

## AI / LLM Providers

### OpenAI
- **Packages**: `openai`
- **Env vars**: `OPENAI_API_KEY`, `OPENAI_ORG_ID`, `OPENAI_BASE_URL`
- **Cost model**: Per-token (input + output priced separately)
- **Pricing**:
  - GPT-4o: $2.50/1M input, $10.00/1M output
  - GPT-4o mini: $0.15/1M input, $0.60/1M output
  - GPT-4.1: $2.00/1M input, $8.00/1M output
  - GPT-4.1 mini: $0.40/1M input, $1.60/1M output
  - GPT-4.1 nano: $0.10/1M input, $0.40/1M output
  - o3: $10.00/1M input, $40.00/1M output
  - o4-mini: $1.10/1M input, $4.40/1M output
- **Rate limits (Tier 1)**: 500 RPM, 200K TPM
- **Rate limits (Tier 2)**: 5,000 RPM, 2M TPM
- **Rate limits (Tier 3)**: 5,000 RPM, 10M TPM
- **Gotchas**: Rate limits are per-organization. Tier upgrades require cumulative spend ($5 -> Tier 1, $50 -> Tier 2, $100 -> Tier 3). Batch API is 50% cheaper but async.

### Anthropic
- **Packages**: `@anthropic-ai/sdk`
- **Env vars**: `ANTHROPIC_API_KEY`
- **Cost model**: Per-token
- **Pricing**:
  - Claude Opus 4: $15.00/1M input, $75.00/1M output
  - Claude Sonnet 4: $3.00/1M input, $15.00/1M output
  - Claude Haiku 3.5: $0.80/1M input, $4.00/1M output
- **Rate limits (Tier 1)**: 50 RPM, 40K input TPM, 8K output TPM
- **Rate limits (Tier 2)**: 1,000 RPM, 80K input TPM, 16K output TPM
- **Rate limits (Tier 3)**: 2,000 RPM, 160K input TPM, 32K output TPM
- **Rate limits (Tier 4)**: 4,000 RPM, 400K input TPM, 80K output TPM
- **Gotchas**: Tier upgrades based on deposit ($5 -> Tier 1, $40 -> Tier 2, $200 -> Tier 3, $400 -> Tier 4). Extended thinking uses output tokens. Prompt caching reduces input cost by 90%.

### Google Gemini
- **Packages**: `@google/generative-ai`, `@google/genai`
- **Env vars**: `GEMINI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_GENERATIVE_AI_API_KEY`
- **Cost model**: Per-token (free tier available)
- **Free tier**: 15 RPM, 1M tokens/minute, 1,500 requests/day
- **Pricing (pay-as-you-go)**:
  - Gemini 2.5 Flash: $0.15/1M input, $0.60/1M output (thinking: $3.50/1M)
  - Gemini 2.5 Pro: $1.25/1M input, $10.00/1M output
  - Gemini 2.0 Flash: $0.10/1M input, $0.40/1M output
- **Rate limits (paid)**: 2,000 RPM, 4M TPM
- **Gotchas**: Free tier rate limits are very restrictive for production. Thinking tokens on 2.5 models cost significantly more than regular output.

### Groq
- **Packages**: `groq-sdk`
- **Env vars**: `GROQ_API_KEY`
- **Cost model**: Per-token (generous free tier)
- **Free tier**: 30 RPM, 15K tokens/min, 14.4K requests/day
- **Pricing**:
  - Llama 3.3 70B: $0.59/1M input, $0.79/1M output
  - Llama 3.1 8B: $0.05/1M input, $0.08/1M output
  - Mixtral 8x7B: $0.24/1M input, $0.24/1M output
- **Gotchas**: Very fast inference but limited model selection. Free tier has daily limits that reset.

### Mistral
- **Packages**: `@mistralai/mistralai`
- **Env vars**: `MISTRAL_API_KEY`
- **Cost model**: Per-token
- **Pricing**:
  - Mistral Large: $2.00/1M input, $6.00/1M output
  - Mistral Small: $0.10/1M input, $0.30/1M output
  - Codestral: $0.30/1M input, $0.90/1M output
- **Rate limits**: Varies by plan. Free: 1 RPM, 500K tokens/month.

### Replicate
- **Packages**: `replicate`
- **Env vars**: `REPLICATE_API_TOKEN`
- **Cost model**: Per-second of compute time
- **Pricing**: Varies by model and hardware. Typical: $0.00025/sec (CPU) to $0.0023/sec (A100 GPU)
- **Gotchas**: Cold starts can add 5-30s. Pricing depends on hardware the model runs on, not just the model.

### Vercel AI SDK (meta-framework)
- **Packages**: `ai`, `@ai-sdk/openai`, `@ai-sdk/anthropic`, `@ai-sdk/google`, `@ai-sdk/mistral`
- **Detection note**: The `ai` package is the SDK. Check for `@ai-sdk/*` provider packages to identify which LLM is actually used. Cost follows the underlying provider's pricing.

---

## Database / BaaS

### Supabase
- **Packages**: `@supabase/supabase-js`, `@supabase/ssr`, `supabase`
- **Env vars**: `SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_ANON_KEY`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_DB_URL`, `DATABASE_URL` (containing `.supabase.`)
- **Cost model**: Flat monthly + usage overages
- **Tiers**:
  - Free: 500 MB database, 1 GB file storage, 50K MAU, 500K edge function invocations, 5 GB bandwidth, 2 projects
  - Pro ($25/mo): 8 GB database, 100 GB storage, 100K MAU, 2M edge function invocations, 250 GB bandwidth
  - Team ($599/mo): 8 GB database, 100 GB storage, 100K MAU, priority support
- **Overages (Pro)**:
  - Database: $0.125/GB beyond 8 GB
  - Bandwidth: $0.09/GB beyond 250 GB
  - Storage: $0.021/GB beyond 100 GB
  - MAU: $0.00325/user beyond 100K
- **Rate limits**: No hard API rate limit, but connection pooling matters. Free tier: 60 connections. Pro: 200 connections.
- **Gotchas**: Pauses inactive free-tier projects after 7 days. RPC functions count as edge invocations. Realtime connections are limited. Database size includes indexes.

### Firebase
- **Packages**: `firebase`, `firebase-admin`
- **Env vars**: `FIREBASE_API_KEY`, `FIREBASE_PROJECT_ID`, `FIREBASE_PRIVATE_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`
- **Cost model**: Pay-per-operation (Blaze plan)
- **Tiers**:
  - Spark (free): 1 GB Firestore storage, 50K reads/day, 20K writes/day, 20K deletes/day, 50K auth MAU
  - Blaze (pay-as-you-go):
    - Firestore: $0.06/100K reads, $0.18/100K writes, $0.02/100K deletes
    - Storage: $0.026/GB, $0.12/GB download
    - Auth: free up to 50K MAU, then phone auth $0.01-0.06/verification
- **Gotchas**: Reads can explode with listeners and real-time subscriptions. Each document field update counts as a write. Firestore indexes count toward storage.

### Neon (Serverless Postgres)
- **Packages**: `@neondatabase/serverless`
- **Env vars**: `DATABASE_URL` (containing `.neon.tech`), `NEON_DATABASE_URL`
- **Cost model**: Compute hours + storage
- **Tiers**:
  - Free: 0.5 GB storage, 191 compute hours/month (shared)
  - Launch ($19/mo): 10 GB storage, 300 compute hours
  - Scale ($69/mo): 50 GB storage, 750 compute hours, autoscaling, read replicas
- **Gotchas**: Cold starts on free tier (can add 1-3s). Compute hours accumulate even with idle connections.

### PlanetScale
- **Packages**: `@planetscale/database`
- **Env vars**: `DATABASE_URL` (containing `.psdb.cloud`)
- **Cost model**: Flat monthly (no free tier since April 2024)
- **Tiers**:
  - Scaler Pro ($39/mo): 10 GB storage, 1B row reads/month, 50M row writes/month
  - Enterprise: Custom
- **Gotchas**: No free tier. MySQL-compatible only. No foreign keys by default (use planetscale-specific tooling).

### MongoDB Atlas
- **Packages**: `mongodb`, `mongoose`
- **Env vars**: `MONGODB_URI`, `MONGO_URL`, `DATABASE_URL` (containing `mongodb`)
- **Cost model**: Cluster-based
- **Tiers**:
  - M0 (free forever): 512 MB storage, shared RAM, 100 max connections
  - M10 (~$57/mo): 10 GB storage, 2 GB RAM, 1,500 connections
  - M20 (~$170/mo): 20 GB storage, 4 GB RAM, 3,000 connections
- **Gotchas**: Free tier has no backups, limited performance. Atlas Search is additional cost.

### Upstash Redis
- **Packages**: `@upstash/redis`
- **Env vars**: `UPSTASH_REDIS_REST_URL`, `UPSTASH_REDIS_REST_TOKEN`, `KV_REST_API_URL`
- **Cost model**: Per-command or fixed
- **Tiers**:
  - Free: 10K commands/day, 256 MB
  - Pay-as-you-go: $0.2/100K commands
  - Fixed ($10/mo): 250 MB, unlimited commands (within reason)
- **Gotchas**: REST-based (HTTP per command), higher latency than native Redis. Daily command limit on free tier resets at midnight UTC.

### Turso (SQLite edge)
- **Packages**: `@libsql/client`
- **Env vars**: `TURSO_DATABASE_URL`, `TURSO_AUTH_TOKEN`
- **Cost model**: Row reads/writes + storage
- **Tiers**:
  - Starter (free): 9 GB storage, 500 databases, 24B row reads/month, 24M row writes/month
  - Scaler ($29/mo): 24 GB, additional reads/writes at metered cost
- **Gotchas**: SQLite limitations apply (limited concurrent writes). Edge replicas add latency for writes.

---

## Email Services

### Resend
- **Packages**: `resend`
- **Env vars**: `RESEND_API_KEY`, `EMAIL_FROM`
- **Cost model**: Per-email
- **Tiers**:
  - Free: 3,000 emails/month, 100 emails/day
  - Pro ($20/mo): 50,000 emails/month
  - Business ($80/mo): 200,000 emails/month
  - Enterprise: Custom
- **Rate limits**: 10 emails/second (free), 100/second (pro)
- **Gotchas**: Daily limit on free tier (100/day) can block you even before monthly limit. Verified domain required for production.

### SendGrid
- **Packages**: `@sendgrid/mail`, `@sendgrid/client`
- **Env vars**: `SENDGRID_API_KEY`
- **Cost model**: Per-email
- **Tiers**:
  - Free tier discontinued (July 2025)
  - Essentials ($19.95/mo): 50,000 emails/month
  - Pro ($89.95/mo): 100,000 emails/month
- **Rate limits**: Varies by plan
- **Gotchas**: No more free tier. Email validation is a separate paid feature.

### Postmark
- **Packages**: `postmark`
- **Env vars**: `POSTMARK_API_TOKEN`, `POSTMARK_SERVER_TOKEN`
- **Cost model**: Per-email
- **Tiers**:
  - Free: 100 test emails/month
  - $15/mo: 10,000 emails/month
  - $50/mo: 50,000 emails/month
  - $100/mo: 125,000 emails/month
- **Gotchas**: Strict anti-spam policies. Separate streams for transactional vs marketing.

### AWS SES
- **Packages**: `@aws-sdk/client-ses`, `aws-sdk` (v2), `nodemailer` (with SES transport)
- **Env vars**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `SES_FROM_EMAIL`
- **Cost model**: Per-email (cheapest at scale)
- **Pricing**: $0.10/1,000 emails ($0.0001/email). Attachments: $0.12/GB
- **Rate limits**: Starts at 200/day (sandbox), request increase for production (typically 50K/day)
- **Gotchas**: Must move out of sandbox for production. No built-in templates (bring your own). Bounce/complaint handling is your responsibility.

### Mailgun
- **Packages**: `mailgun.js`, `mailgun-js`
- **Env vars**: `MAILGUN_API_KEY`, `MAILGUN_DOMAIN`
- **Cost model**: Per-email
- **Tiers**:
  - Flex (free-ish): 100 emails/day for first 3 months
  - Foundation ($15/mo): 50,000 emails/month
  - Scale ($60/mo): 100,000 emails/month
- **Gotchas**: Free tier is a trial, not permanent. Must use their subdomain or verify your own.

---

## Hosting / Compute

### Vercel
- **Detection**: `vercel.json`, `.vercel/` directory, `vercel` in devDependencies
- **Env vars**: `VERCEL`, `VERCEL_URL`, `VERCEL_ENV`
- **Cost model**: Flat monthly + usage overages
- **Tiers**:
  - Hobby (free): 100 GB bandwidth, 100K function invocations, 10s function timeout, 1 concurrent build
  - Pro ($20/mo per member): 1 TB bandwidth, 1M function invocations, 300s function timeout, 3 concurrent builds
  - Enterprise: Custom
- **Overages (Pro)**: $40/100 GB bandwidth, $0.60/1M additional invocations
- **Gotchas**: 10s function timeout on Hobby kills any non-trivial backend work. `maxDuration` requires Pro. Cron jobs: 1/day on Hobby, every 1 min on Pro. Edge functions have 128 MB memory limit. `after()` runs count toward function duration.

### Netlify
- **Detection**: `netlify.toml`, `_redirects`, `_headers`
- **Cost model**: Flat monthly + usage
- **Tiers**:
  - Free: 100 GB bandwidth, 125K function invocations, 300 build minutes/month
  - Pro ($19/mo per member): 1 TB bandwidth, 2M function invocations
- **Gotchas**: Function timeout 10s (free) / 26s (pro). Background functions up to 15 min (pro only).

### Fly.io
- **Detection**: `fly.toml`
- **Env vars**: `FLY_APP_NAME`, `FLY_REGION`
- **Cost model**: Per-VM + bandwidth
- **Tiers**:
  - Free: 3 shared-cpu-1x VMs (256 MB each), 160 GB bandwidth
  - Pay-as-you-go: shared-cpu-1x $1.94/mo, dedicated-cpu-1x $5.70/mo
- **Gotchas**: Free tier VMs auto-stop after inactivity. Cold starts possible. Volumes are region-specific.

### Railway
- **Detection**: `railway.json`, `railway.toml`
- **Cost model**: Usage-based (compute + memory)
- **Tiers**:
  - Trial: $5 one-time credit, 30-day expiry
  - Hobby ($5/mo): $5 included usage, then metered
  - Pro ($20/mo): Team features, priority support
- **Pricing**: vCPU $0.000463/min, Memory $0.000231/GB/min
- **Gotchas**: No permanent free tier. Trial expires. Persistent volumes cost extra.

### Render
- **Detection**: `render.yaml`
- **Cost model**: Per-instance
- **Tiers**:
  - Free: Static sites only (web services removed from free tier)
  - Starter ($7/mo): 512 MB RAM, 0.5 CPU
  - Standard ($25/mo): 2 GB RAM, 1 CPU
- **Gotchas**: Free web services were removed. 15-min spin-down on starter. Databases are separate pricing.

### AWS EC2
- **Detection**: AWS SDK packages, terraform with `aws_instance`, GitHub Actions with AWS deploy
- **Env vars**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `EC2_HOST`, `SSH_KEY`
- **Cost model**: Per-hour (on-demand, reserved, spot)
- **Common instances**:
  - t3.micro: $0.0104/hr (~$7.50/mo) - 2 vCPU, 1 GB RAM
  - t3.small: $0.0208/hr (~$15/mo) - 2 vCPU, 2 GB RAM
  - t3.medium: $0.0416/hr (~$30/mo) - 2 vCPU, 4 GB RAM
  - t3.large: $0.0832/hr (~$60/mo) - 2 vCPU, 8 GB RAM
- **Savings**: Reserved instances save 30-60%. Spot instances save up to 90% but can be interrupted.
- **Gotchas**: You pay for the instance even when idle. EBS storage, data transfer, and elastic IPs are additional costs. t3 instances have burstable CPU (credit system).

### Cloudflare Workers
- **Detection**: `wrangler.toml`
- **Env vars**: `CF_ACCOUNT_ID`, `CF_API_TOKEN`
- **Cost model**: Per-request
- **Tiers**:
  - Free: 100K requests/day, 10ms CPU time
  - Paid ($5/mo): 10M requests/month, 30s CPU time
- **Gotchas**: 128 MB memory limit. No native Node.js APIs (Workers runtime). KV/D1/R2 have separate pricing.

---

## Auth Providers

### Auth0
- **Packages**: `@auth0/nextjs-auth0`, `auth0`, `@auth0/auth0-react`
- **Env vars**: `AUTH0_DOMAIN`, `AUTH0_CLIENT_ID`, `AUTH0_CLIENT_SECRET`, `AUTH0_AUDIENCE`
- **Cost model**: Per-MAU
- **Tiers**:
  - Free: 7,500 MAU, 2 social connections
  - Essentials ($35/mo): 500 MAU included, +$0.07/MAU
  - Professional ($240/mo): 1,000 MAU included, +$0.07/MAU, MFA, custom domains
  - Enterprise: Custom
- **Gotchas**: Expensive at scale. MAU pricing means even inactive users returning once count. Machine-to-machine tokens are separately limited.

### Clerk
- **Packages**: `@clerk/nextjs`, `@clerk/clerk-sdk-node`, `@clerk/express`
- **Env vars**: `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_WEBHOOK_SECRET`
- **Cost model**: Per-MAU
- **Tiers**:
  - Free: 10,000 MAU, 5 social connections
  - Pro ($25/mo): 10K MAU included, +$0.02/MAU after
  - Enterprise: Custom
- **Gotchas**: Clerk JS bundle adds to page weight. MAU includes any user who authenticates, even if they don't use the app that month.

### Kinde
- **Packages**: `@kinde-oss/kinde-auth-nextjs`, `@kinde/kinde-node-express`
- **Env vars**: `KINDE_DOMAIN`, `KINDE_CLIENT_ID`, `KINDE_CLIENT_SECRET`
- **Cost model**: Per-MAU
- **Tiers**:
  - Free: 10,500 MAU
  - Pro ($25/mo): 10.5K MAU included, +$0.035/MAU
- **Gotchas**: Relatively new, smaller ecosystem.

### NextAuth / Auth.js
- **Packages**: `next-auth`, `@auth/core`
- **Env vars**: `NEXTAUTH_SECRET`, `NEXTAUTH_URL`
- **Cost model**: Free (self-hosted, open source)
- **Gotchas**: You manage sessions, database adapters, and security yourself. OAuth provider rate limits still apply.

---

## Payments

### Stripe
- **Packages**: `stripe`, `@stripe/stripe-js`, `@stripe/react-stripe-js`
- **Env vars**: `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_WEBHOOK_SECRET`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`
- **Cost model**: Per-transaction (percentage)
- **Pricing**:
  - Standard: 2.9% + $0.30 per successful card charge
  - International cards: +1.5%
  - Currency conversion: +1%
  - Stripe Billing (subscriptions): +0.5-0.8%
  - Stripe Tax: +0.5%
  - Stripe Radar (fraud): $0.05/screened or $0.07/screened with manual review
- **Gotchas**: Fees stack (card + international + billing + tax can be 5%+). Chargebacks cost $15 each. Payout timing is 2 days (US).

### Paddle
- **Packages**: `@paddle/paddle-node-sdk`, `@paddle/paddle-js`
- **Env vars**: `PADDLE_API_KEY`, `PADDLE_CLIENT_TOKEN`
- **Cost model**: Per-transaction (merchant of record)
- **Pricing**: 5% + $0.50 per transaction
- **Gotchas**: Higher percentage but handles all tax, compliance, invoicing. Not available in all countries.

### LemonSqueezy
- **Packages**: `@lemonsqueezy/lemonsqueezy.js`
- **Env vars**: `LEMONSQUEEZY_API_KEY`, `LEMONSQUEEZY_STORE_ID`
- **Cost model**: Per-transaction (merchant of record)
- **Pricing**: 5% + $0.50 per transaction
- **Gotchas**: Similar to Paddle. Good for digital products. Limited payment methods compared to Stripe.

---

## Storage

### AWS S3
- **Packages**: `@aws-sdk/client-s3`, `aws-sdk`
- **Env vars**: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `S3_BUCKET`, `AWS_S3_BUCKET`
- **Cost model**: Storage + requests + egress
- **Pricing**:
  - Storage: $0.023/GB/month (standard)
  - PUT/POST: $0.005/1,000 requests
  - GET: $0.0004/1,000 requests
  - Data transfer out: $0.09/GB (first 10 TB)
- **Free tier (12 months)**: 5 GB storage, 20K GET, 2K PUT
- **Gotchas**: Egress costs add up fast. Consider CloudFront CDN in front to reduce direct S3 egress.

### Cloudflare R2
- **Packages**: `@aws-sdk/client-s3` (S3-compatible), `wrangler`
- **Env vars**: `R2_BUCKET`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `CF_ACCOUNT_ID`
- **Cost model**: Storage + operations (ZERO egress)
- **Pricing**:
  - Storage: $0.015/GB/month
  - Class A (writes): $4.50/million
  - Class B (reads): $0.36/million
  - Egress: FREE
- **Free tier**: 10 GB storage, 1M Class A, 10M Class B per month
- **Gotchas**: S3-compatible but not identical. Some S3 features missing. Great for egress-heavy workloads.

### Cloudinary
- **Packages**: `cloudinary`, `next-cloudinary`
- **Env vars**: `CLOUDINARY_URL`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
- **Cost model**: Credits-based (transformations + storage + bandwidth)
- **Tiers**:
  - Free: 25 monthly credits (~25K transformations or 25 GB bandwidth)
  - Plus ($89/mo): 225 credits
  - Advanced ($224/mo): 600 credits
- **Gotchas**: Credit system is confusing. Different operations cost different credit amounts. Easy to burn credits with on-the-fly transformations.

### Uploadthing
- **Packages**: `uploadthing`, `@uploadthing/react`
- **Env vars**: `UPLOADTHING_SECRET`, `UPLOADTHING_APP_ID`
- **Cost model**: Storage-based
- **Tiers**:
  - Free: 2 GB storage
  - Paid ($10/mo): 100 GB storage
- **Gotchas**: Built for Next.js/React. Limited to file uploads (not a general object store).

---

## Analytics & Monitoring

### PostHog
- **Packages**: `posthog-js`, `posthog-node`
- **Env vars**: `NEXT_PUBLIC_POSTHOG_KEY`, `POSTHOG_API_KEY`, `POSTHOG_HOST`
- **Cost model**: Per-event
- **Tiers**:
  - Free: 1M events/month, 5K session recordings, 1M feature flag requests
  - Pay-as-you-go: ~$0.00045/event after free tier
- **Gotchas**: Events can explode with autocapture enabled. Session recordings are separately metered. Self-hosted option available (free, you pay infra).

### Mixpanel
- **Packages**: `mixpanel`, `mixpanel-browser`
- **Env vars**: `MIXPANEL_TOKEN`, `NEXT_PUBLIC_MIXPANEL_TOKEN`
- **Cost model**: Per-event
- **Tiers**:
  - Free: 1M events/month
  - Growth ($28/mo): 10K MTU included
- **Gotchas**: MTU (monthly tracked users) pricing on Growth plan, not events.

### Sentry
- **Packages**: `@sentry/nextjs`, `@sentry/node`, `@sentry/react`, `@sentry/browser`
- **Env vars**: `SENTRY_DSN`, `NEXT_PUBLIC_SENTRY_DSN`, `SENTRY_AUTH_TOKEN`
- **Cost model**: Per-event
- **Tiers**:
  - Developer (free): 5K errors/month, 10K performance transactions, 50 session replays
  - Team ($26/mo): 50K errors, 100K transactions, 500 replays
  - Business ($80/mo): 50K errors, 100K transactions, 500 replays, advanced features
- **Overages**: $0.000290/error, $0.000290/transaction after included
- **Gotchas**: Performance monitoring can generate massive event volumes. Source maps upload in CI adds build time.

### Datadog
- **Packages**: `dd-trace`, `datadog-metrics`
- **Env vars**: `DD_API_KEY`, `DD_APP_KEY`, `DD_SITE`
- **Cost model**: Per-host + per-feature
- **Pricing**:
  - Infrastructure: $15/host/month (Pro), $23/host/month (Enterprise)
  - APM: $31/host/month
  - Logs: $0.10/GB ingested, $1.70/million log events
- **Gotchas**: Extremely expensive at scale. Each feature (infra, APM, logs, synthetics) is separately priced. Costs can 10x unexpectedly.

---

## Cache / Queue / Background Jobs

### Upstash QStash
- **Packages**: `@upstash/qstash`
- **Env vars**: `QSTASH_TOKEN`, `QSTASH_URL`
- **Cost model**: Per-message
- **Tiers**:
  - Free: 500 messages/day
  - Pay-as-you-go ($1/mo base): 100K messages/month included
- **Gotchas**: HTTP-based messaging. Good for serverless but higher latency than dedicated queues.

### Inngest
- **Packages**: `inngest`
- **Env vars**: `INNGEST_EVENT_KEY`, `INNGEST_SIGNING_KEY`
- **Cost model**: Per-function-run
- **Tiers**:
  - Free: 50K function runs/month
  - Team ($50/mo): 500K runs, priority execution
- **Gotchas**: Great for durable workflows but each step in a multi-step function counts as a separate run.

### Trigger.dev
- **Packages**: `@trigger.dev/sdk`
- **Env vars**: `TRIGGER_API_KEY`, `TRIGGER_API_URL`
- **Cost model**: Per-run
- **Tiers**:
  - Free: 50K runs/month, 5 concurrent
  - Pro ($30/mo): 250K runs, 25 concurrent
- **Gotchas**: Runs have a max duration. Concurrent limits can cause queuing at scale.

### BullMQ (self-hosted with Redis)
- **Packages**: `bullmq`, `bull`
- **Env vars**: `REDIS_URL`, `REDIS_HOST`
- **Cost model**: Free (you pay for Redis hosting)
- **Gotchas**: Requires a Redis instance. If using Upstash Redis, be aware of command limits. Production Redis (AWS ElastiCache, Redis Cloud) has its own pricing.

---

## Messaging / Communication

### Twilio
- **Packages**: `twilio`
- **Env vars**: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER`
- **Cost model**: Per-message / per-minute
- **Pricing**:
  - SMS: $0.0079/message (US)
  - Voice: $0.0085/min (US)
  - WhatsApp: $0.005/message + Meta conversation fees
  - Verify (OTP): $0.05/verification
- **Gotchas**: International SMS is much more expensive ($0.05-0.15+). Phone number rental is $1.15/month. Verify costs add up fast with phone auth.

### Svix (Webhooks)
- **Packages**: `svix`
- **Env vars**: `SVIX_API_KEY`, `SVIX_WEBHOOK_SECRET`, `WEBHOOK_SECRET`
- **Cost model**: Per-message
- **Tiers**:
  - Free: 50K messages/month
  - Starter ($50/mo): 500K messages
  - Business ($250/mo): 2.5M messages
- **Detection note**: Sometimes used only for webhook verification (consumer side), not sending. Check if the app is sending webhooks or just receiving/verifying them. Receiving-only has no Svix cost.

---

## Hosting Detection Heuristics

Config file -> Platform mapping:
```
vercel.json or .vercel/           -> Vercel
fly.toml                          -> Fly.io
Dockerfile + docker-compose*.yml  -> Self-hosted (EC2, VPS, etc.)
railway.json or railway.toml      -> Railway
netlify.toml                      -> Netlify
render.yaml                       -> Render
wrangler.toml                     -> Cloudflare Workers
serverless.yml or serverless.ts   -> AWS Lambda (Serverless Framework)
terraform/*.tf                    -> Parse provider blocks
cdk.json                          -> AWS CDK
sam.yaml or template.yaml         -> AWS SAM
.github/workflows/*.yml           -> Check for deploy steps to identify platform
Procfile                          -> Heroku
app.yaml                          -> Google App Engine
```

## ORM -> Database Provider Detection

```
prisma/schema.prisma -> read "provider" field:
  "postgresql" -> Supabase, Neon, or generic Postgres (check DATABASE_URL)
  "mysql"      -> PlanetScale or generic MySQL
  "sqlite"     -> Turso or local SQLite
  "mongodb"    -> MongoDB Atlas
  "cockroachdb" -> CockroachDB

drizzle.config.ts -> read "dialect" field:
  "postgresql" -> check connection string for provider
  "mysql"      -> check connection string
  "sqlite"     -> check if using Turso driver
```

## Common Env Var -> Service Patterns

When you see env vars without matching packages, use these patterns:
```
*_API_KEY          -> Service name is usually the prefix
DATABASE_URL       -> Parse the connection string for host
REDIS_URL          -> Parse for host (upstash, redis-cloud, etc.)
*_WEBHOOK_SECRET   -> Indicates webhook integration
NEXT_PUBLIC_*      -> Client-exposed config
*_DSN              -> Typically Sentry
CRON_SECRET        -> Cron job authentication
JWT_SECRET         -> JWT-based auth (self-managed)
SESSION_SECRET     -> Session-based auth
```
