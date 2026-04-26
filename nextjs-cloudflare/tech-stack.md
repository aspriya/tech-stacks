# Tech Stack: Next.js + Cloudflare for Small SaaS

> **Purpose of this document.** This is a reference for AI coding agents (Devin, Claude Code, Cursor agents, etc.) working on small-to-mid scale SaaS products. It documents the canonical tech stack and explains *what each piece does* and *when to choose it*. Agents should consult this when scaffolding new components, picking storage, or making architectural decisions. When multiple options exist for a use case, the agent should pick based on the "When to use" guidance — not invent alternatives.
>
> **Versions and dates referenced are current as of April 2026.**

---

## 0. Stack at a Glance

| Layer | Choice | Why |
|---|---|---|
| Frontend framework | Next.js (App Router) | Mature React framework, RSC + Server Actions, large ecosystem |
| Runtime / hosting | Cloudflare Workers via `@opennextjs/cloudflare` | Single platform for frontend + backend, zero egress, global edge |
| API layer | Next.js Route Handlers + Server Actions | Co-located with UI, no separate backend service needed |
| Primary database | Cloudflare D1 (SQLite, serverless) | Native to Workers, scales horizontally, cheap |
| Object storage | Cloudflare R2 | S3-compatible, **zero egress fees** |
| Cache / sessions / config | Cloudflare KV | Fast global reads, simple key-value |
| Real-time / per-tenant state | Cloudflare Durable Objects | WebSockets, presence, multiplayer, strict consistency |
| Background jobs | Cloudflare Queues + Workflows | Decoupled async processing, retries, multi-step orchestration |
| Authentication | Better Auth (default) or Clerk (managed) | See §6 for decision criteria |
| Vector search / RAG | Cloudflare Vectorize | Native, integrates with Workers AI |
| AI inference | Workers AI + AI Gateway | Edge GPU inference, observability and fallback |
| Email | Resend | Simple API, Workers-friendly, good DX |
| ORM | Drizzle | First-class D1 support, type-safe, lightweight |

---

## 1. Frontend & Application Framework

### Next.js (App Router)

The application framework. Use the **App Router** (`app/` directory), not the legacy Pages Router.

**Use it for:**
- All UI pages, layouts, and React components
- Server Components for data fetching close to the database
- Client Components only when you need browser-only APIs or interactivity
- Server Actions for form mutations and small write operations
- Route Handlers (`app/api/*/route.ts`) for REST-style endpoints, webhooks, and any API consumed by external clients

**Rules for the agent:**
- Default to Server Components. Mark client islands explicitly with `"use client"`.
- Do **not** add `export const runtime = "edge"` to any file. The OpenNext adapter does not support the Next.js Edge runtime — the entire app already runs in the Cloudflare Workers Node.js-compatible runtime.
- Keep route handlers thin — they should validate input, call a service/data-access layer, and return a response. Business logic lives in `lib/` modules.
- Use `next.config.ts` (TypeScript) over `next.config.js`.

---

## 2. Hosting & Deployment: Cloudflare Workers + OpenNext

The Next.js app is deployed to **Cloudflare Workers** using the **`@opennextjs/cloudflare`** adapter. This is the official, supported path as of 2026 — Cloudflare Pages with `next-on-pages` is **deprecated** for full-stack Next.js apps; the Cloudflare docs themselves now redirect Next.js users to the Workers guide.

### How it works

OpenNext takes the standard `next build` output (in `standalone` mode) and transforms it into a single Worker bundle that runs on Cloudflare's global network. Static assets are served via Workers Static Assets.

### Required configuration

- `@opennextjs/cloudflare` as a dependency
- `wrangler` as a devDependency
- `wrangler.jsonc` with `nodejs_compat` flag and a recent `compatibility_date`
- `open-next.config.ts` at project root
- Add `.open-next/` to `.gitignore`

### Standard scripts in `package.json`

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "preview": "opennextjs-cloudflare build && opennextjs-cloudflare preview",
    "deploy": "opennextjs-cloudflare build && opennextjs-cloudflare deploy",
    "cf-typegen": "wrangler types --env-interface CloudflareEnv cloudflare-env.d.ts"
  }
}
```

### Local dev with bindings

In `next.config.ts`, call `initOpenNextCloudflareForDev()` so `next dev` exposes local versions of D1, R2, KV, etc.

### Bindings, not env vars, for Cloudflare resources

Cloudflare resources (D1, R2, KV, Queues, Durable Objects, Vectorize, AI) are accessed via **bindings** declared in `wrangler.jsonc`, not connection strings. In server code, get them via:

```ts
import { getCloudflareContext } from "@opennextjs/cloudflare";
const { env } = getCloudflareContext();
// env.DB, env.MY_BUCKET, env.CACHE, etc.
```

Run `npm run cf-typegen` after changing bindings to regenerate types.

### CI/CD

Use **Workers Builds** (Cloudflare's native CI) or GitHub Actions running `wrangler deploy`. Workers Builds free plan caps at 3,000 build minutes/month; paid plan includes 6,000 minutes plus $0.005/minute thereafter, with 6 concurrent builds.

---

## 3. API Patterns

There is no separate backend service. The "backend" is the same Next.js app, running on the same Worker. Three API surfaces exist:

### 3a. Server Components
For reads that render UI. Fetch data directly from D1/KV/R2 inside async Server Components. No HTTP round-trip needed.

### 3b. Server Actions
For form submissions and small mutations triggered by the UI. Co-locate with the component. Always validate input with Zod.

### 3c. Route Handlers (`app/api/.../route.ts`)
Use for:
- Webhooks (Resend, Clerk, etc.)
- Endpoints consumed by third parties or mobile clients
- Public REST/JSON APIs
- Auth provider callback routes

**Pattern: a route handler should be ~20 lines.** Validate → call a function from `lib/` → return JSON. Heavy logic does not live in route files.

---

## 4. Data Layer: Picking the Right Storage

This is the most important decision tree. The agent must pick deliberately, not default to one option.

### 4a. D1 — Primary Relational Database

SQLite-based, serverless, queryable from Workers with near-zero latency. Each database is capped at **10 GB**, but you can have **up to 50,000 databases per account** — D1 is designed for horizontal scale-out (per-tenant, per-customer, per-entity databases).

**Use D1 for:**
- Core application data: users, organizations, projects, posts
- Anything you'd put in Postgres for a typical SaaS
- Read-heavy workloads (most web apps)
- Multi-tenant data where each tenant fits comfortably under 10 GB

**Pricing (Workers Paid plan, $5/mo includes):**
- 25 billion rows read/month
- 50 million rows written/month
- 5 GB storage
- Beyond: $0.001 per million rows read, $1.00 per million rows written, $0.75/GB-month
- **Zero egress fees**

**ORM:** Use **Drizzle**. It has first-class D1 support, is lightweight, type-safe, and works well with the Workers runtime. Prisma also works but is heavier.

**Watch out for:**
- D1 reads via replicas can have stale-read issues for write-then-immediate-read patterns. Use Sessions API for read-after-write consistency when needed.
- Indexes matter — full table scans count every row read against your quota.
- 10 GB ceiling per database. Plan tenant sharding if a single tenant might exceed that.

### 4b. KV — Cache, Sessions, Config

Eventually-consistent global key-value store. Reads are fast and cheap (cached at the edge); writes propagate within ~60 seconds and are limited to 1 write/sec per key.

**Use KV for:**
- Session tokens (when not using a managed auth provider)
- Feature flags, runtime configuration
- API keys, rate-limit counters (for low-precision limits)
- Caching computed responses, public profile data
- Anything read very frequently and rarely written

**Do not use KV for:**
- Anything needing immediate consistency
- Anything that needs queries beyond `get`/`put`/`list` by prefix
- Anything modified more than once per second per key

### 4c. R2 — Object Storage

S3-compatible object storage with **zero egress fees**. Same use cases as S3, plus saves money on bandwidth-heavy workloads.

**Use R2 for:**
- User-uploaded files (avatars, documents, attachments)
- Generated assets (PDFs, exports, AI-generated images)
- Backups and archives
- Static media for the app

**Pricing:** $0.015/GB-month storage, plus per-operation costs for Class A (writes) and Class B (reads). No egress charges.

**Pattern for uploads:** Use **presigned URLs** so clients upload directly to R2 instead of through the Worker (avoids the Worker's CPU and request size limits).

### 4d. Durable Objects — Stateful Coordination

Singleton objects with their own SQLite-backed storage and the ability to maintain in-memory state across requests. Provide strict serializability — perfect for things that need a single source of truth.

**Use Durable Objects for:**
- WebSocket connections (chat, collaboration, presence)
- Per-room or per-document state in real-time apps
- Per-tenant rate limiting or counters that need precision
- Coordination primitives (locks, queues, leader election)
- Any state where "exactly one place writes this" matters

**Do not use Durable Objects for:**
- Bulk relational data (use D1)
- Storing files (use R2)
- Eventually-consistent caches (use KV)

### 4e. Hyperdrive — Connect to External Postgres/MySQL

If the SaaS requires Postgres specifically (e.g., pgvector, PostGIS, complex queries D1 can't handle, or a database larger than 10 GB), use **Hyperdrive** to connect Workers to an existing Postgres or MySQL hosted on Neon, Supabase, PlanetScale, AWS RDS, etc. Hyperdrive handles connection pooling and edge query caching.

**Use Hyperdrive when:**
- The app already has a Postgres database
- Single-database size will exceed 10 GB
- Need Postgres-specific features (extensions, complex SQL, etc.)

**Default to D1 first.** Only reach for Hyperdrive when D1 genuinely doesn't fit.

### 4f. Vectorize — Vector Database

Cloudflare's globally-replicated vector database. Pairs natively with Workers AI for RAG, semantic search, recommendations.

**Use Vectorize for:**
- RAG pipelines (chat-with-your-docs)
- Semantic search across user content
- Recommendation systems based on embeddings
- Anomaly detection

**Limit:** Best when corpus is under ~5M vectors. For very large or filter-heavy workloads, consider external (Qdrant, pgvector via Hyperdrive).

---

## 5. Async Work: Queues & Workflows

### 5a. Cloudflare Queues

Managed message queue with at-least-once delivery and zero egress charges. A producer Worker enqueues; a consumer Worker processes batches.

**Use Queues for:**
- Email sending after signup
- Image/video processing after upload
- Webhook fan-out
- Anything that should not block a user request
- Decoupling slow third-party calls from request paths

Always configure a **dead letter queue** for failed messages.

### 5b. Cloudflare Workflows

Durable, multi-step orchestration with automatic retries and resumability. Each step is checkpointed — if a step fails, only that step retries.

**Use Workflows for:**
- Multi-step business processes (signup → create org → seed data → send welcome email)
- Long-running jobs that span minutes/hours
- Anything where partial progress must survive failures
- Saga-style coordinated multi-service operations

**Queues vs Workflows:** Queues are for fire-and-forget messages. Workflows are for ordered, multi-step processes that need durability.

### 5c. Cron Triggers

For scheduled tasks (daily reports, cleanup jobs, data retention sweeps). Defined in `wrangler.jsonc`, executed by the Workers runtime — no separate scheduler service.

---

## 6. Authentication

Pick **one** of these. Do not mix.

### Default: Better Auth

Self-hosted, MIT-licensed, code-first auth library. As of 2026 it is the recommended default for new self-hosted Next.js projects — Auth.js maintenance was effectively handed off to the Better Auth team in late 2025, and Auth.js is now in security-patch mode.

**Use Better Auth when:**
- Cost matters (no per-MAU fees)
- You need data ownership (sessions in your D1)
- You want built-in 2FA, passkeys, organizations/multi-tenancy without paying extra
- You're fine running auth as part of the same Worker

**Caveat:** Better Auth's full session validation expects a Node-compatible runtime. The OpenNext + `nodejs_compat` setup handles this fine, but middleware-based session checks should be configured carefully.

### Alternative: Clerk

Hosted auth with pre-built UI components. Pick this when speed-to-launch beats cost and data-ownership.

**Use Clerk when:**
- The product is B2C and time-to-launch is critical
- You want polished pre-built UI (sign-in, user profile, org switcher) for free
- Edge middleware session checks are required and cannot tolerate Node.js
- The product needs organization/team management with minimal setup

**Pricing (Feb 2026):** Free up to 10,000 MAU, then ~$0.02/MAU. At 50K MAU this is ~$800/month — meaningful but predictable.

### Alternative: WorkOS AuthKit

Pick this only when **enterprise SSO/SCIM/Directory Sync** is on the near-term roadmap. Free up to 1M MAU for user management. Overkill for most early-stage SaaS.

### Decision rule for the agent

1. Building a B2B SaaS that will sell to enterprises within 12 months → **WorkOS**
2. Need ship-it-this-week speed and budget is fine → **Clerk**
3. Everything else → **Better Auth**

---

## 7. AI / LLM Features (Optional)

Only include this layer if the product has AI features. Skip otherwise.

### Workers AI
Run open-source models (Llama, Mistral, Whisper, embedding models, rerankers, etc.) on Cloudflare's serverless GPUs. Edge inference, no GPU management.

**Use for:** embeddings, transcription, classification, lightweight chat.

### AI Gateway
A proxy in front of any LLM provider (OpenAI, Anthropic, Workers AI). Adds caching, rate-limiting, retries, model fallback, observability, cost analytics.

**Use for:** any production LLM call. Always route through AI Gateway, even for OpenAI/Anthropic — you get observability and cost control for free.

### Architecture pattern (RAG)
1. **Ingestion:** Worker validates upload → R2 → enqueue Queue message
2. **Processing:** Consumer Worker chunks doc → Workers AI for embeddings → Vectorize upsert
3. **Query:** Worker takes user question → embed → Vectorize search → assemble context → call LLM (via AI Gateway) → stream response

Keep CPU-heavy work (re-ranking, large parsing) **off the request path** — Workers' CPU budget is the real constraint.

---

## 8. Supporting Services

### Email — Resend
Transactional email. Simple HTTP API, plays well with Workers, React Email for templates.

### Observability
- **Workers Logs / Tail** for live request logs
- **Workers Analytics Engine** for custom time-series metrics (per-tenant usage, feature adoption)
- **AI Gateway** for LLM call observability
- Optional: Sentry for error tracking (works fine on Workers)

### Validation — Zod
All inputs (route handlers, server actions, env vars) validated with Zod schemas. No exceptions.

### Forms — React Hook Form + Zod
Standard combo. Server-side validation with the same Zod schema is mandatory; client-side validation alone is never sufficient.

### UI components — shadcn/ui + Tailwind CSS
Default UI primitives. Copy-paste components, full ownership, no runtime dependency.

---

## 9. Cost Model (Rough Guidance)

| App stage | Expected monthly Cloudflare cost |
|---|---|
| Hobby / under 100K req/day | **Free** |
| Small SaaS, 1–5M req/month, modest D1+KV+R2 | **~$15–50** |
| Mid-traffic SaaS with Durable Objects, real-time, larger D1 | **~$50–200** |
| Auth via Clerk at 10K MAU | **+$0**; at 50K MAU: **+~$800** |

The big wins vs AWS/Vercel: **zero egress fees** (R2, D1), **no per-region pricing**, **single $5 base** covering compute and most data services.

The biggest gotcha: **CPU time, not request count, is usually the limit**. The included 30M CPU-ms/month sounds generous but a single CPU-heavy worker (image processing, large JSON parsing) can burn through it fast. Offload CPU-bound work to Queues, Workers AI, or external services.

---

## 10. Project Structure (Reference)

```
my-saas/
├── app/                          # Next.js App Router
│   ├── (marketing)/              # Public marketing pages
│   ├── (app)/                    # Authenticated app
│   ├── api/                      # Route Handlers (webhooks, public APIs)
│   └── layout.tsx
├── components/                   # React components (shadcn/ui in components/ui)
├── lib/
│   ├── db/                       # Drizzle schema + client
│   ├── auth/                     # Better Auth config
│   ├── services/                 # Business logic (used by routes & actions)
│   ├── queues/                   # Queue producers and consumer handlers
│   └── workflows/                # Workflow definitions
├── workers/                      # Standalone Workers (queue consumers, cron, DOs)
├── drizzle/                      # Migrations
├── public/
├── wrangler.jsonc                # Cloudflare bindings
├── open-next.config.ts
├── next.config.ts
└── package.json
```

---

## 11. Decision Heuristics for the Agent

When a feature needs a backend component, walk this list in order:

1. **Does it need to render UI?** → Server Component or Client Component in `app/`.
2. **Is it a form mutation tied to a UI?** → Server Action.
3. **Is it called by a webhook, third party, or mobile client?** → Route Handler in `app/api/`.
4. **Does it need to persist relational data?** → D1 + Drizzle.
5. **Is the data a file/blob?** → R2 (with presigned URLs for uploads).
6. **Is it a high-read, low-write key lookup (sessions, config, flags)?** → KV.
7. **Does it need WebSockets, presence, or per-entity strict consistency?** → Durable Object.
8. **Is it slow and shouldn't block the request?** → Queue.
9. **Is it multi-step with retries and durability requirements?** → Workflow.
10. **Is it on a schedule?** → Cron Trigger.
11. **Does it need vector/semantic search?** → Vectorize (+ Workers AI for embeddings).
12. **Does it need an LLM call?** → AI Gateway in front of the model provider.

If none of the above clearly fits, surface the ambiguity rather than picking arbitrarily.

---

## 12. Things the Agent Should NOT Do

- Do **not** add `export const runtime = "edge"` to any Next.js route or page.
- Do **not** use `localStorage`/`sessionStorage` in Server Components or Server Actions.
- Do **not** call external Postgres/MySQL directly from a Worker without going through Hyperdrive.
- Do **not** hold open database connections at module scope assuming they persist — Workers are stateless.
- Do **not** put long-running work (>a few seconds of CPU) on the request path. Use Queues or Workflows.
- Do **not** put secrets in `wrangler.jsonc` — use `wrangler secret put` and access via `env`.
- Do **not** mix Cloudflare Pages and Workers deployment paths. The project deploys to Workers via OpenNext.
- Do **not** introduce a separate Express/Hono backend service unless explicitly requested. Route Handlers + Server Actions cover the API surface.
- Do **not** add an ORM other than Drizzle without a stated reason. Prisma is acceptable if explicitly requested; otherwise default to Drizzle.
- Do **not** implement custom auth from scratch — use Better Auth, Clerk, or WorkOS per §6.

---

*Last updated: April 2026. Verify Cloudflare pricing and product status at https://developers.cloudflare.com before committing to any tier-sensitive decision.*
