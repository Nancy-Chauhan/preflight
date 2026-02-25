---
name: preflight
description: >
  Simulate what happens to your system at scale. Reads your code, traces every
  service call, and tells you exactly where things break - rate limits, hard caps,
  cost cliffs, timing issues, cascading failures. Use when a user asks about
  deployment costs, scaling limits, service dependencies, migration planning,
  infrastructure mapping, or "can my stack handle X users?" Also triggers on
  "preflight", "preflight check", "launch readiness", or "what will this cost?"
argument-hint: "[user-count | 'map' | blank for full report]"
---

# Preflight: Operational Simulation for Any Codebase

You are running a preflight check on this project. Your job is to read the actual code, trace every external service call, and simulate exactly what happens when real users hit this system. Do not guess or use generic benchmarks. Every number you report must come from what you find in the code.

## Step 0: Parse Arguments

Interpret `$ARGUMENTS`:
- **No arguments or blank**: Run the full report. Simulate at 100, 1,000, and 10,000 monthly active users.
- **A number** (e.g., `500`): Simulate at that user count. Also show a 10x projection.
- **`map`**: Skip cost/simulation. Only produce the infrastructure dependency map.

---

## Phase 1: Service Discovery

Scan the project to find every external service, dependency, and infrastructure component. Do all of these steps:

### 1.1 Package Manager Scan
Read the dependency manifest:
- **Node.js**: `package.json` (check both `dependencies` and `devDependencies`)
- **Python**: `requirements.txt`, `pyproject.toml`, `Pipfile`
- **Go**: `go.mod`
- **Ruby**: `Gemfile`
- **Rust**: `Cargo.toml`

Cross-reference every dependency against the service detection table in `services.md`. Record each match with the exact package name and version.

### 1.2 Environment Variable Scan
Read all env files in priority order: `.env.example`, `.env.local`, `.env`, `.env.production`, `.env.development`. For each env var:
- Identify which service it belongs to (use patterns in `services.md`)
- Note if it contains URLs, API keys, connection strings, or secrets
- Parse connection strings for provider hints (e.g., `DATABASE_URL` containing `.supabase.` or `.neon.tech`)

### 1.3 Infrastructure Config Scan
Check for these files and extract deployment/hosting info:
- `vercel.json` or `.vercel/` directory -> Vercel
- `fly.toml` -> Fly.io
- `Dockerfile`, `docker-compose*.yml` -> Self-hosted / containerized
- `railway.json`, `railway.toml` -> Railway
- `netlify.toml` -> Netlify
- `render.yaml` -> Render
- `wrangler.toml` -> Cloudflare Workers
- `serverless.yml`, `serverless.ts` -> AWS Lambda (Serverless Framework)
- `terraform/*.tf` -> Parse for provider blocks and resources
- `cdk.json`, `sam.yaml` -> AWS CDK / SAM
- `.github/workflows/*.yml` -> CI/CD pipeline, look for deploy targets

### 1.4 ORM & Database Detection
If an ORM is detected:
- **Prisma**: Read `prisma/schema.prisma`, extract the `provider` field (postgresql, mysql, sqlite, mongodb, cockroachdb)
- **Drizzle**: Read `drizzle.config.ts` or `drizzle.config.js`, check the `dialect` field
- **TypeORM**: Check `ormconfig.json` or `data-source.ts` for database type
- **Sequelize**: Check config files for dialect

### 1.5 Import Pattern Scan
Use Grep to search source files for import patterns of services detected in 1.1. Count how many files import each service. This reveals how deeply integrated each service is.

### 1.6 CI/CD and DevOps Scan
Read CI/CD configs for:
- Deploy targets and environments
- Scheduled jobs / cron triggers
- Secret names (reveal additional services)
- Docker registry usage
- CDN or edge deployment config

**Output of Phase 1**: A table of every detected service with:
| Service | Category | Detection Evidence | Env Vars | Files Using It |

---

## Phase 2: Flow Tracing & Multiplier Detection

This phase is critical. You must trace HOW services are called, not just THAT they exist.

### 2.1 Find All Entry Points
Identify every way work enters the system:

- **API Routes**: In Next.js: `app/api/**/route.ts`. In Express: look for `app.get/post/put/delete`. In Django: `urls.py`. In Rails: `routes.rb`. In FastAPI: decorated functions.
- **Cron Jobs / Scheduled Tasks**: Look for cron config in `vercel.json`, `fly.toml`, GitHub Actions schedules, or cron libraries (`node-cron`, `cron`, `@inngest/inngest`).
- **Webhooks**: API routes that verify signatures (look for `svix`, HMAC verification, webhook secret usage).
- **Queue Consumers**: Look for BullMQ workers, SQS consumers, Inngest functions, Trigger.dev tasks.
- **Pages / Client-Side**: Pages that make API calls on load or user interaction.

### 2.2 Trace Each Entry Point
For each entry point found, read the file and follow the import chain:
1. What functions does it call?
2. What do those functions import?
3. Which service clients get invoked?
4. Build the full chain: `Entry Point -> Function -> Service Call`

### 2.3 Detect Multipliers
For each flow, look for patterns that multiply service calls:
- **Loops over users**: `.map()`, `.forEach()`, `for...of` over user lists, `Promise.all()`, `Promise.allSettled()`
- **Batch operations**: Processing arrays of items, paginated fetches
- **Concurrency limits**: Look for concurrency constants, semaphores, `p-limit`, custom batching
- **Per-item processing**: "for each article, call LLM" or "for each user, send email"

Record the multiplier for each flow: "1 per user", "1 per article", "1 per cron run", "N per batch".

### 2.4 Detect Hard Caps & Limits in Code
Search for constants and limits that constrain throughput:
- `.limit(N)` in database queries
- `MAX_*`, `LIMIT_*`, `BATCH_SIZE`, `CONCURRENCY_*` constants
- `maxDuration`, `timeout` settings in route configs
- Rate limiter configurations (requests per minute/second)
- Retry limits and backoff constants
- Queue batch sizes

### 2.5 Detect Rate Limiters
Look for rate limiting implementations:
- In-memory rate limiters (Map-based -- flag as broken for multi-instance)
- Redis-based rate limiters (works across instances)
- API route rate limit decorators or middleware
- External service rate limits mentioned in comments or constants

### 2.6 Classify Flows by Scaling Behavior
Categorize each flow:
- **Fixed cost**: Runs on schedule regardless of user count (cron jobs, RSS ingestion)
- **Linear with users**: Scales 1:1 with user count (signup emails, daily newsletters per user)
- **Superlinear**: Grows faster than user count (notification fan-out, feed generation)
- **Triggered externally**: Webhook volume depends on external service (incoming emails, payment events)

**Output of Phase 2**: A flow table:
```
| Trigger | Flow Chain | Services Called | Multiplier | Frequency | Scaling |
```

---

## Phase 3: Operational Simulation

Using the flows from Phase 2, simulate a realistic day and month at the target user count.

### 3.1 Daily Simulation Narrative
Write a narrative describing what happens in a typical day. Be specific with numbers:

> "Each hour, your ingest cron fires. It processes 57 RSS feeds in batches of 5,
> making up to 285 LLM categorization calls (1,024 max tokens each). This takes
> approximately 19 minutes at 15 RPM on Gemini free tier.
>
> At 8 AM in each user's timezone, the newsletter cron fires. With 500 users spread
> across timezones, roughly 60 users are due per hour. At concurrency 5 with ~3s per
> newsletter (LLM call + email render + send), that's 36 seconds per batch of 5,
> or ~7 minutes for 60 users. Well within the 300s function timeout.
>
> Each signup triggers: 1 Supabase write, 1 LLM profile parse (4,096 max tokens),
> 1 Resend confirmation email. At 5 signups/day, that's 5 LLM calls + 5 emails."

### 3.2 Monthly Usage Projection
For each detected service, calculate monthly usage:

```
| Service | Resource | Monthly Usage | Current Tier Limit | % Used | Status |
```

### 3.3 Full Tier Analysis
For each service, show the COMPLETE tier picture:
- What tier are they likely on now?
- How much of that tier does projected usage consume?
- At what user count does the current tier max out?
- What does the next tier cost?
- What's the limit on that next tier?
- Where's the cliff after that?

Format:
```
### Resend
Current tier: Free (3,000 emails/month)
Projected usage at 500 users: ~15,500 emails/month
Free tier maxes out at: ~97 users
Next tier: Pro ($20/month) - 50,000 emails/month
Pro tier maxes out at: ~1,600 users
After that: Business - custom pricing
```

### 3.4 Cost Summary Table
```
| Service | 100 users | 500 users | 1,000 users | 10,000 users |
| (cost)  | (cost)    | (cost)    | (cost)      | (cost)       |
| TOTAL   |           |           |             |              |
```

### 3.5 Data Growth Projection
Estimate storage and data growth:
- Database rows added per day/month (from insert operations found in code)
- File storage growth (if uploads exist)
- Log volume growth
- Index size implications

---

## Phase 4: Bottleneck & Breaking Point Analysis

This is where you find the things that will actually break. For each finding, reference the specific file and line where the issue lives.

### Categories of Breaking Points

**Hard Caps in Code**
Look for limits in the code that will cause silent failures or data loss at scale:
- Query limits that skip users (e.g., `LIMIT 100` means user 101 gets nothing)
- Batch sizes that can't process all items within timeout
- Queue fetch limits that leave items unprocessed

**Rate Limits**
Calculate whether the system can complete its work within rate limits:
- External API rate limits (LLM providers, email services)
- Self-imposed rate limits in code
- Time to complete a batch at the rate limit vs. available time window

**Timing Conflicts**
Do the math on whether operations finish in time:
- Total processing time = (items / concurrency) x time-per-item
- Compare against: function timeout, cron interval, user expectations
- Flag when processing time exceeds available window

**Cascading Failures**
Trace what happens when a service call fails:
- Does one failure stop the entire batch?
- Are there retries? How many? With what backoff?
- Does a failure in service A cause service B to also fail?
- Is there error recovery or does the user just miss out?

**Concurrency & Multi-Instance Issues**
- In-memory state that breaks with multiple instances (rate limiters, caches, locks)
- Database-level locking that might cause contention
- Race conditions in optimistic locking patterns

**Data Growth Issues**
- Tables that grow without bounds (no cleanup/archival)
- Indexes that degrade with size (GIN, full-text)
- Connection pool exhaustion at scale

### Severity Levels
- **PASS**: No issue at the target user count
- **WARN**: Will become an issue soon, or has a workaround but is fragile
- **FAIL**: Will break at the target user count, or is already broken

Format each finding as:
```
[FAIL] Newsletter cron hard cap
  File: src/lib/supabase/queries.ts:42
  Issue: get_users_due_for_newsletter() has LIMIT 100. At 150+ users,
         50 users will not receive their newsletter on any given day.
  Impact: Silent data loss - users don't get newsletters, no error logged.
  Fix: Implement pagination or increase limit with corresponding
       timeout/batch adjustments.
```

---

## Phase 5: Infrastructure Dependency Map

### 5.1 Env Var Dependency Graph
For each env var, map: Variable -> Service -> Files that use it.
Use Grep to count references across the codebase.

### 5.2 Domain & URL Chain
Trace how the app's domain/URL connects services:
- DNS provider -> points to hosting IP/URL
- Hosting -> serves the app at the domain
- OAuth providers -> callback URLs containing the domain
- BaaS (Supabase, Firebase) -> redirect URLs, site URL config
- Email service -> verified sender domain
- Webhook registrations -> endpoint URLs at the domain
- CORS origins -> allowed domains
- SSL/TLS certificates -> domain coverage

### 5.3 "If You Change X" Table
For each infrastructure node, list what else needs updating:

```
| If You Change...        | Also Update...                                    |
|-------------------------|---------------------------------------------------|
| Server / hosting        | DNS records, OAuth callbacks, webhook URLs,        |
|                         | Supabase site URL, SSL cert, CORS origins          |
| Domain name             | DNS, OAuth callbacks, email verified domain,       |
|                         | webhook URLs, Supabase config, SSL cert,           |
|                         | hardcoded URLs in code, CORS origins               |
| Database                | DATABASE_URL, connection strings, ORM config,      |
|                         | migration state, seed data                         |
| Auth provider           | Login flow, callback routes, session handling,     |
|                         | user table schema                                  |
```

### 5.4 Migration Pitfall Flags
For common migration scenarios, flag specific risks:
- **Vercel to self-hosted**: Cron jobs need external scheduler, `after()` won't work, edge functions need replacement, env vars need manual setup
- **Changing domain**: Every OAuth provider needs updated, DNS propagation delay, email domain re-verification
- **Switching database**: Migration scripts, stored procedures/RPCs, full-text search differences, index strategy changes

---

## Output Format

Structure the final report exactly like this:

```markdown
# Preflight Report: [Project Name]
## Target: [N] monthly active users

---

## Stack Detected

| Service | Category | Evidence | Current Tier | Files |
|---------|----------|----------|--------------|-------|

---

## System Simulation: A Day at [N] Users

[Write the narrative here. Be specific. Use actual numbers from the code.]

---

## Resource Flows

| Trigger | Flow | Services Hit | Multiplier | Frequency |
|---------|------|--------------|------------|-----------|

---

## Resource Usage & Cost Projection

### [Service Name]
Current tier: [tier] ([limit])
Projected usage: [amount]
Tier analysis: [when current maxes out, next tier cost, next cliff]

[Repeat for each service]

### Monthly Cost Summary

| Service | [tier 1] users | [tier 2] users | [tier 3] users |
|---------|----------------|----------------|----------------|
| TOTAL   |                |                |                |

---

## Bottlenecks & Breaking Points

[PASS/WARN/FAIL items with file references and specific numbers]

---

## Infrastructure Dependency Map

[ASCII dependency graph]

### Change Impact Table

| If You Change... | Also Update... |
|------------------|----------------|

---

## Recommendations

### Before Launch (Fix These)
[FAIL items with suggested fixes]

### Monitor Closely
[WARN items with thresholds to watch]

### Can Wait
[Items that only matter at higher scale]
```

---

## Important Guidelines

1. **Every number must come from the code.** Do not use generic benchmarks. If you find `CONCURRENCY_LIMIT = 5` in the code, use 5. If you find `.limit(100)`, use 100.
2. **Read the actual service client files.** Don't just detect the package; read how it's configured and called.
3. **Follow the full chain.** An API route that calls a helper that calls a service client counts as that route using that service.
4. **Be specific about WHERE.** Always include file paths and line numbers for findings.
5. **Distinguish between "costs money" and "breaks."** A WARN for cost is different from a FAIL for "users won't get their data."
6. **Check for missing things too.** No error handling on a critical path? No retry on an API call that can fail? No monitoring? Flag it.
7. **Use `services.md` as your reference** for detection patterns and pricing data. If a service isn't listed there, use WebSearch to find current pricing.
8. **Read `examples.md`** if you need to calibrate the level of detail and tone of the output.
