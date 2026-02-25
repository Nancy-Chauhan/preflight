---
name: preflight
description: >
  Simulate what happens to your system at scale. Reads your code, traces every
  service call, and tells you exactly where things break - rate limits, hard caps,
  cost cliffs, timing issues, cascading failures. Use when a user asks about
  deployment costs, scaling limits, service dependencies, migration planning,
  infrastructure mapping, or "can my stack handle X users?" Also triggers on
  "preflight", "preflight check", "launch readiness", or "what will this cost?"
argument-hint: "[user-count | 'map'] [--export md|pdf|html|all]"
---

# Preflight: Operational Simulation for Any Codebase

You are running a preflight check on this project. Your job is to read the actual code, trace every external service call, and simulate exactly what happens when real users hit this system. Do not guess or use generic benchmarks. Every number you report must come from what you find in the code.

## Step 0: Parse Arguments

Interpret `$ARGUMENTS`:
- **No arguments or blank**: Run the full report. Simulate at 100, 1,000, and 10,000 monthly active users.
- **A number** (e.g., `500`): Simulate at that user count. Also show a 10x projection.
- **`map`**: Skip cost/simulation. Only produce the infrastructure dependency map.
- **`--export <format>`**: After displaying the inline report, also save the report to file(s). Supported formats:
  - `md` -- Save as `./preflight-report.md`
  - `html` -- Save as `./preflight-report.html`
  - `pdf` -- Save as `./preflight-report.pdf` (requires HTML to be generated first)
  - `all` -- Save all three formats (MD, HTML, PDF)

The `--export` flag is additive and can be combined with any other argument. The inline report always displays regardless of export flags. Examples: `/preflight 500 --export md`, `/preflight map --export all`, `/preflight --export pdf`.

---

## Phase 1: Service Discovery

Scan the project to find every external service, dependency, and infrastructure component. Do all of these steps:

### 1.1 Monorepo Detection
First, check if this is a monorepo:
- Look for workspace definitions in `package.json` (`workspaces` field), `pnpm-workspace.yaml`, `turbo.json`, `nx.json`, or `lerna.json`
- If monorepo: find ALL `package.json` files across workspaces and scan each. Track which workspace each service belongs to (e.g., "OpenAI detected in `worker/`, Clerk detected in `backend/` and `frontend/`")
- This matters because different workspaces may use different services and have different scaling characteristics

### 1.2 Package Manager Scan
Read the dependency manifest(s):
- **Node.js**: `package.json` (check both `dependencies` and `devDependencies`). In monorepos, scan root AND all workspace package.json files.
- **Python**: `requirements.txt`, `pyproject.toml`, `Pipfile`
- **Go**: `go.mod`
- **Ruby**: `Gemfile`
- **Rust**: `Cargo.toml`

Cross-reference every dependency against the service detection table in `services.md`. Record each match with the exact package name and version.

### 1.3 Environment Variable Scan
Read all env files in priority order: `.env.example`, `.env.local`, `.env`, `.env.production`, `.env.development`. In monorepos, check env files in each workspace directory too. For each env var:
- Identify which service it belongs to (use patterns in `services.md`)
- Note if it contains URLs, API keys, connection strings, or secrets
- Parse connection strings for provider hints (e.g., `DATABASE_URL` containing `.supabase.` or `.neon.tech`)

### 1.4 Infrastructure Config Scan
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

### 1.5 Self-Hosted Infrastructure Detection
When you find `docker-compose*.yml` files, read them carefully. Services running as Docker containers are SELF-HOSTED -- their cost is server RAM/CPU/disk, NOT SaaS pricing. For each Docker service:
- Record what it is (PostgreSQL, Redis, Kafka/Redpanda, OpenSearch, MinIO, etc.)
- Note its resource settings (memory limits, JVM heap, CPU allocation, volume mounts)
- Mark it as "Self-hosted" in the tier column, not a SaaS tier
- Calculate its cost as a fraction of the server's total cost

Common self-hosted services to watch for:
- `postgres` / `mysql` / `mariadb` -> Database (cost = server RAM + disk)
- `redis` / `valkey` -> Cache (cost = server RAM)
- `redpanda` / `kafka` / `confluent` -> Event streaming (cost = server RAM + disk, CPU-intensive)
- `opensearch` / `elasticsearch` -> Search/analytics (cost = JVM heap, very RAM-hungry)
- `minio` -> S3-compatible storage (cost = disk)
- `temporal` / `temporal-ui` -> Workflow engine (cost = server RAM)
- `grafana` / `loki` / `prometheus` -> Observability (cost = server RAM + disk)
- `nginx` / `traefik` / `caddy` -> Reverse proxy (cost = minimal)
- `rabbitmq` / `nats` -> Message broker (cost = server RAM)

When self-hosted infra is detected, the cost projection should estimate TOTAL SERVER REQUIREMENTS (RAM, CPU, disk) rather than per-service SaaS pricing.

### 1.6 ORM & Database Detection
If an ORM is detected:
- **Prisma**: Read `prisma/schema.prisma`, extract the `provider` field (postgresql, mysql, sqlite, mongodb, cockroachdb)
- **Drizzle**: Read `drizzle.config.ts` or `drizzle.config.js`, check the `dialect` field
- **TypeORM**: Check `ormconfig.json` or `data-source.ts` for database type
- **Sequelize**: Check config files for dialect

### 1.7 Import Pattern Scan
Use Grep to search source files for import patterns of services detected in 1.1. Count how many files import each service. This reveals how deeply integrated each service is.

### 1.8 CI/CD and DevOps Scan
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

- **API Routes**: In Next.js: `app/api/**/route.ts`. In Express/NestJS: look for controllers, `app.get/post/put/delete`, `@Get()/@Post()` decorators. In Django: `urls.py`. In Rails: `routes.rb`. In FastAPI: decorated functions.
- **Cron Jobs / Scheduled Tasks**: Look for cron config in `vercel.json`, `fly.toml`, GitHub Actions schedules, or cron libraries (`node-cron`, `cron`, `@inngest/inngest`).
- **Webhooks**: API routes that verify signatures (look for `svix`, HMAC verification, webhook secret usage).
- **Queue Consumers**: Look for BullMQ workers, SQS consumers, Inngest functions, Trigger.dev tasks, Kafka consumers (`kafkajs`), AMQP consumers (`amqplib`).
- **Workflow Engines**: Look for Temporal workflows (`@temporalio/*`), Inngest functions, Step Functions. These orchestrate multi-step operations and have their own concurrency/timeout settings.
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

### 3.0 Cost Ownership Classification
Before projecting costs, classify WHO pays for each service:

- **Platform cost** (operator pays): Hosting, database, self-hosted infra, SaaS subscriptions (Clerk, PostHog, etc.). These scale with total users and are paid by whoever runs the platform.
- **User-borne cost** (end users pay): When users bring their own API keys (common for LLM-powered tools, cloud integrations). The platform operator doesn't pay -- each user pays their own provider directly.
- **Per-use cost** (operator pays per user action): Transaction fees (Stripe), per-email costs (Resend), metered APIs. Operator pays but proportional to usage.

This distinction is critical. A platform where users provide their own OpenAI keys has very different economics than one where the operator pays for all LLM calls. Report costs in separate sections when both types exist.

### 3.0.1 Self-Hosted Infrastructure Sizing
When the project uses self-hosted Docker services, calculate total server requirements:
- Sum RAM needs across all containers (with headroom)
- Consider CPU-intensive services (search indexing, event streaming, PDF generation)
- Estimate disk growth per month
- Recommend specific instance types (e.g., "t3.large: 8 GB RAM, $60/mo" or "t3.xlarge: 16 GB RAM, $120/mo")
- Note services that will need more resources at scale (OpenSearch heap, Kafka/Redpanda cores)

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

**Secret & Encryption Key Risks**
- Encryption master keys where rotation would break all existing encrypted data
- Session secrets where changing them invalidates all active sessions
- JWT secrets where changing them invalidates all issued tokens
- Default/placeholder secrets in env examples that might be used in production (e.g., `CHANGE_ME_32_CHAR_SECRET_KEY!!!!`)

**Phantom Dependencies**
- Packages in package.json that are imported by a framework transitively but never used directly in code (e.g., `mqtt` pulled in by `@nestjs/microservices` but no MQTT code exists). Flag these as "detected but unused" to avoid confusion.

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

### 5.4 Inter-Service Dependencies (Self-Hosted)
When self-hosted Docker services are detected, map their internal dependency chain:
- Which containers depend on which (via `depends_on`, connection strings)
- Startup order requirements (DB must be healthy before app starts)
- Shared volumes and data directories
- Network topology (which ports are exposed externally vs internal-only)
- Services that share the same database (e.g., app DB and Temporal DB in same PostgreSQL instance)

### 5.5 Migration Pitfall Flags
For common migration scenarios, flag specific risks:
- **Vercel to self-hosted**: Cron jobs need external scheduler, `after()` won't work, edge functions need replacement, env vars need manual setup
- **Self-hosted to cloud managed**: Docker compose services need SaaS replacements (PostgreSQL -> Supabase/Neon, Redis -> Upstash, MinIO -> S3), each with different connection strings and auth
- **Changing domain**: Every OAuth provider needs updated, DNS propagation delay, email domain re-verification
- **Switching database**: Migration scripts, stored procedures/RPCs, full-text search differences, index strategy changes
- **Changing encryption/secret keys**: Flag if any master encryption key change would render existing data unreadable

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

## Phase 6: Export Report

**Only run this phase if `--export` was specified in the arguments.**

After displaying the full inline report, export it to the requested format(s). The export order is always: Markdown first, then HTML, then PDF (since PDF depends on HTML).

### 6.1 Markdown Export (`--export md` or `--export all`)

Write the report content to `./preflight-report.md` using the Write tool. Use the exact same Markdown content that was displayed inline -- no modifications needed.

### 6.2 HTML Export (`--export html`, `--export pdf`, or `--export all`)

Generate a self-contained HTML file at `./preflight-report.html`. The HTML must be a single file with all CSS embedded (no external stylesheets, no JavaScript dependencies).

Use this exact HTML/CSS template structure, filling in the report data:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Preflight Report: [Project Name]</title>
<style>
  @page { margin: 1.5cm; size: A4; }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
    font-size: 11px;
    line-height: 1.5;
    color: #1a1a2e;
    background: #fff;
    padding: 40px;
    max-width: 900px;
    margin: 0 auto;
  }
  h1 {
    font-size: 28px;
    font-weight: 800;
    color: #0f0f23;
    margin-bottom: 4px;
    letter-spacing: -0.5px;
  }
  .subtitle {
    font-size: 14px;
    color: #6b7280;
    margin-bottom: 8px;
  }
  .target-badge {
    display: inline-block;
    background: #2563eb;
    color: #fff;
    padding: 4px 14px;
    border-radius: 20px;
    font-size: 13px;
    font-weight: 600;
    margin-bottom: 24px;
  }
  .meta-row {
    display: flex;
    gap: 24px;
    margin-bottom: 24px;
    flex-wrap: wrap;
  }
  .meta-item {
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    padding: 10px 16px;
    flex: 1;
    min-width: 180px;
  }
  .meta-item .label { font-size: 10px; color: #6b7280; text-transform: uppercase; letter-spacing: 0.5px; }
  .meta-item .value { font-size: 16px; font-weight: 700; color: #0f0f23; }
  h2 {
    font-size: 18px;
    font-weight: 700;
    color: #0f0f23;
    margin-top: 32px;
    margin-bottom: 12px;
    padding-bottom: 6px;
    border-bottom: 2px solid #2563eb;
  }
  h3 {
    font-size: 14px;
    font-weight: 700;
    color: #1e293b;
    margin-top: 20px;
    margin-bottom: 8px;
  }
  h4 {
    font-size: 12px;
    font-weight: 600;
    color: #334155;
    margin-top: 14px;
    margin-bottom: 6px;
  }
  p { margin-bottom: 8px; }
  table {
    width: 100%;
    border-collapse: collapse;
    margin-bottom: 16px;
    font-size: 10.5px;
  }
  th {
    background: #f1f5f9;
    color: #334155;
    font-weight: 600;
    text-align: left;
    padding: 6px 8px;
    border-bottom: 2px solid #cbd5e1;
    font-size: 10px;
    text-transform: uppercase;
    letter-spacing: 0.3px;
  }
  td {
    padding: 5px 8px;
    border-bottom: 1px solid #e2e8f0;
    vertical-align: top;
  }
  tr:hover { background: #f8fafc; }
  .finding {
    border-left: 4px solid;
    padding: 10px 14px;
    margin-bottom: 12px;
    border-radius: 0 6px 6px 0;
    background: #fafafa;
    page-break-inside: avoid;
  }
  .finding.critical { border-color: #7c2d12; background: #fef2f2; }
  .finding.fail { border-color: #dc2626; background: #fef2f2; }
  .finding.warn { border-color: #f59e0b; background: #fffbeb; }
  .finding.pass { border-color: #16a34a; background: #f0fdf4; }
  .finding .badge {
    display: inline-block;
    padding: 1px 8px;
    border-radius: 4px;
    font-size: 10px;
    font-weight: 700;
    letter-spacing: 0.5px;
    margin-right: 6px;
    vertical-align: middle;
  }
  .finding.critical .badge { background: #7c2d12; color: #fff; }
  .finding.fail .badge { background: #dc2626; color: #fff; }
  .finding.warn .badge { background: #f59e0b; color: #fff; }
  .finding.pass .badge { background: #16a34a; color: #fff; }
  .finding .title { font-weight: 700; font-size: 12px; }
  .finding .file { font-family: 'SF Mono', 'Fira Code', monospace; font-size: 10px; color: #6b7280; margin-top: 4px; }
  .finding .desc { margin-top: 4px; font-size: 11px; }
  .finding .fix { margin-top: 4px; font-size: 11px; color: #1e40af; }
  .code-block {
    background: #1e293b;
    color: #e2e8f0;
    padding: 12px 16px;
    border-radius: 6px;
    font-family: 'SF Mono', 'Fira Code', monospace;
    font-size: 10px;
    line-height: 1.6;
    margin-bottom: 16px;
    overflow-x: auto;
    white-space: pre;
  }
  .cost-highlight {
    background: #dbeafe;
    border: 1px solid #93c5fd;
    border-radius: 8px;
    padding: 12px 16px;
    margin-bottom: 16px;
  }
  .cost-highlight .amount { font-size: 24px; font-weight: 800; color: #1d4ed8; }
  .cost-highlight .label { font-size: 11px; color: #3b82f6; }
  .dep-graph {
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 8px;
    padding: 16px;
    font-family: 'SF Mono', 'Fira Code', monospace;
    font-size: 10px;
    line-height: 1.6;
    margin-bottom: 16px;
    white-space: pre;
  }
  .section-intro {
    color: #475569;
    font-size: 11.5px;
    margin-bottom: 12px;
  }
  .two-col {
    display: flex;
    gap: 16px;
    margin-bottom: 16px;
  }
  .two-col > div { flex: 1; }
  .phantom-badge {
    display: inline-block;
    background: #f1f5f9;
    border: 1px solid #cbd5e1;
    color: #64748b;
    padding: 1px 8px;
    border-radius: 4px;
    font-size: 10px;
    font-weight: 600;
  }
  .user-borne {
    display: inline-block;
    background: #ede9fe;
    border: 1px solid #c4b5fd;
    color: #6d28d9;
    padding: 1px 8px;
    border-radius: 4px;
    font-size: 10px;
    font-weight: 600;
  }
  .self-hosted {
    display: inline-block;
    background: #ecfdf5;
    border: 1px solid #86efac;
    color: #15803d;
    padding: 1px 8px;
    border-radius: 4px;
    font-size: 10px;
    font-weight: 600;
  }
  .saas {
    display: inline-block;
    background: #dbeafe;
    border: 1px solid #93c5fd;
    color: #1d4ed8;
    padding: 1px 8px;
    border-radius: 4px;
    font-size: 10px;
    font-weight: 600;
  }
  footer {
    margin-top: 40px;
    padding-top: 16px;
    border-top: 1px solid #e2e8f0;
    font-size: 10px;
    color: #94a3b8;
    text-align: center;
  }
  @media print {
    body { padding: 0; }
    .finding { page-break-inside: avoid; }
    h2 { page-break-after: avoid; }
  }
</style>
</head>
<body>
<!-- Fill in report content here using semantic HTML -->
<!-- Use the CSS classes above for consistent styling -->
<footer>
  Generated by <strong>Preflight</strong> &mdash; github.com/chauhan/preflight<br>
  Report date: [current date]
</footer>
</body>
</html>
```

**HTML content mapping from the Markdown report:**

- **Title**: `<h1>Preflight Report</h1>` with `<div class="subtitle">` for the project description
- **Target user count**: `<span class="target-badge">Target: N Monthly Active Users</span>`
- **Key metrics**: Use `<div class="meta-row">` with `<div class="meta-item">` cards
- **Section headers**: `<h2>` tags (get the blue bottom border automatically)
- **Tables**: Standard `<table>` with `<thead>` and `<tbody>`
- **Findings**: Each finding is a `<div class="finding [severity]">` where severity is `critical`, `fail`, `warn`, or `pass`. Inside use: `<span class="badge">SEVERITY</span>`, `<span class="title">`, `<div class="file">`, `<div class="desc">`, `<div class="fix">`
- **Code blocks / ASCII diagrams**: `<div class="code-block">` for dark background code, `<div class="dep-graph">` for light background dependency graphs
- **Cost highlights**: `<div class="cost-highlight">` with `.amount` and `.label` children
- **Service type badges**: `<span class="self-hosted">Self-hosted</span>`, `<span class="saas">SaaS</span>`, `<span class="user-borne">User-Borne</span>`, `<span class="phantom-badge">PHANTOM</span>`

### 6.3 PDF Export (`--export pdf` or `--export all`)

Generate the HTML file first (if not already generated in 6.2), then convert it to PDF.

Try these tools in order, using the first one found:

1. **Google Chrome**: Run via Bash:
   ```
   "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --disable-gpu --print-to-pdf=./preflight-report.pdf --no-pdf-header-footer ./preflight-report.html
   ```
   Also try `google-chrome` and `google-chrome-stable` as command names for Linux.

2. **Chromium**: Run via Bash:
   ```
   chromium --headless --disable-gpu --print-to-pdf=./preflight-report.pdf --no-pdf-header-footer ./preflight-report.html
   ```
   Also try `chromium-browser`.

3. **wkhtmltopdf**: Run via Bash:
   ```
   wkhtmltopdf --page-size A4 --margin-top 15mm --margin-bottom 15mm --margin-left 15mm --margin-right 15mm ./preflight-report.html ./preflight-report.pdf
   ```

4. **weasyprint**: Run via Bash:
   ```
   weasyprint ./preflight-report.html ./preflight-report.pdf
   ```

If none of these tools are available, tell the user:
> PDF generation requires a headless browser or PDF tool. Install Chrome, Chromium, wkhtmltopdf, or weasyprint. Alternatively, open `./preflight-report.html` in your browser and print to PDF (Cmd+P / Ctrl+P).

If the HTML was generated only as a dependency for PDF (i.e., the user requested `--export pdf` but not `--export html` or `--export all`), keep the HTML file anyway -- it's useful as a fallback.

### 6.4 Export Summary

After all exports complete, print a summary:

```
**Exported files:**
- Markdown: ./preflight-report.md
- HTML: ./preflight-report.html
- PDF: ./preflight-report.pdf
```

Only list the files that were actually created.

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
9. **Distinguish self-hosted from SaaS.** Docker services cost server RAM/CPU, not SaaS fees. Report server sizing, not per-service pricing.
10. **Distinguish who pays.** If users bring their own API keys (LLM, cloud), that's user-borne cost, not platform cost. Report separately.
11. **Flag phantom dependencies.** Packages installed but never imported in code are noise. Note them as "detected but unused" so the report stays accurate.
12. **Monorepos need per-workspace tracking.** In a monorepo, different workspaces (frontend, backend, worker) have different services and different scaling characteristics. Don't lump them together.
