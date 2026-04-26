# Cloudflare Automation: Wrangler CLI & MCP Servers

> **Purpose of this document.** This is a reference for AI coding agents (Devin, Claude Code, GitHub Copilot agents, Cursor, etc.) for **provisioning, configuring, and operating Cloudflare resources** without human dashboard work. It pairs with the tech stack document. The agent should consult this whenever it needs to create a database, deploy a Worker, set a secret, run a migration, or inspect logs.
>
> **Versions and dates referenced are current as of April 2026.**

---

## 0. TL;DR for the Agent

Cloudflare exposes three automation surfaces. Use them in this order of preference:

1. **Wrangler CLI** — the primary tool. Anything in the local repo (deploy, migrations, secrets, resource creation) goes through Wrangler. This is what runs in dev and in CI.
2. **Cloudflare MCP servers** — for natural-language operations and discovery. Useful for questions like "show me my D1 schema," "tail this Worker's logs," or "create a Vectorize index." Ideal when the agent is operating interactively.
3. **Cloudflare REST API** — fallback for things Wrangler doesn't cover, or when scripting from non-Node environments. Use the API MCP server's `execute()` tool rather than constructing requests by hand when possible.

The agent should **never** ask a human to "go click around the Cloudflare dashboard." Everything below can be scripted.

---

## 1. Wrangler — The Primary CLI

Wrangler is Cloudflare's official CLI for the entire developer platform. It is the canonical way to manage Workers, KV, R2, D1, Durable Objects, Queues, Workflows, Vectorize, Hyperdrive, Workers AI, Pipelines, Secrets, and more.

### Install & invocation

Wrangler should be installed **as a project devDependency**, not globally:

```bash
npm i -D wrangler@latest
```

Invoke via the package manager:

```bash
npx wrangler <command>
# or
pnpm wrangler <command>
```

### Authentication

```bash
# Interactive (opens browser) — for local dev
wrangler login

# Headless (CI/CD, agents) — set as env var
export CLOUDFLARE_API_TOKEN=<token>
export CLOUDFLARE_ACCOUNT_ID=<account_id>
```

For agents running in CI: always use `CLOUDFLARE_API_TOKEN`. Never run `wrangler login` from a non-interactive context.

Generate scoped API tokens at the Cloudflare dashboard → My Profile → API Tokens. For agent automation, scope tokens narrowly (e.g., "Workers Scripts: Edit", "D1: Edit", "Workers KV Storage: Edit") instead of using the global key.

### Configuration file

Use `wrangler.jsonc` (preferred — newer features are JSON-only). `wrangler.toml` still works but should be avoided for new projects.

A canonical `wrangler.jsonc` for a Next.js + OpenNext project:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-saas",
  "main": ".open-next/worker.js",
  "compatibility_date": "2026-04-24",
  "compatibility_flags": ["nodejs_compat"],
  "assets": {
    "directory": ".open-next/assets",
    "binding": "ASSETS"
  },
  "vars": {
    "ENVIRONMENT": "production"
  },
  "kv_namespaces": [
    { "binding": "CACHE", "id": "<KV_NAMESPACE_ID>" }
  ],
  "r2_buckets": [
    { "binding": "UPLOADS", "bucket_name": "my-saas-uploads" }
  ],
  "d1_databases": [
    { "binding": "DB", "database_name": "my-saas-db", "database_id": "<DB_ID>" }
  ],
  "queues": {
    "producers": [{ "binding": "EMAIL_QUEUE", "queue": "email-jobs" }],
    "consumers": [{ "queue": "email-jobs", "max_batch_size": 10 }]
  },
  "ai": { "binding": "AI" }
}
```

### Auto-provisioning resources

Wrangler can create KV, R2, and D1 resources at deploy time if their IDs are missing from the config. Add the binding without `id`/`bucket_name`/`database_id`, run `wrangler deploy`, and Wrangler will create the resources, name them with the worker prefix, and write IDs back to the config file.

This is the simplest path for an agent bootstrapping a new project — declare bindings, then deploy.

---

## 2. Wrangler Command Reference (Agent Cheatsheet)

Commands the agent will use most often, grouped by task.

### Project lifecycle

```bash
# Create a new Worker project
npm create cloudflare@latest -- my-app

# Local dev server
wrangler dev
wrangler dev --port 3000
wrangler dev --remote        # Hit real remote bindings (use with care)

# Deploy
wrangler deploy
wrangler deploy --env staging

# Versioned deploys & rollback
wrangler versions list
wrangler rollback [VERSION_ID]
```

### Logs & observability

```bash
# Live tail logs for a deployed Worker
wrangler tail
wrangler tail --env production
wrangler tail --format json
wrangler tail --status error    # Filter by error responses
```

### Secrets

```bash
# Set a production secret (interactive prompt for value)
wrangler secret put RESEND_API_KEY
wrangler secret put DATABASE_URL --env staging

# List secret names (values are not retrievable)
wrangler secret list

# Remove
wrangler secret delete RESEND_API_KEY
```

For local dev, put the same keys in a **`.dev.vars`** file at project root (gitignored). Never commit secrets to `wrangler.jsonc`.

### D1 (database)

```bash
# Create a database (writes ID to config if --binding is provided)
wrangler d1 create my-saas-db

# List, info, delete
wrangler d1 list
wrangler d1 info my-saas-db
wrangler d1 delete my-saas-db

# Run SQL — local
wrangler d1 execute my-saas-db --local --command "SELECT * FROM users LIMIT 5"
wrangler d1 execute my-saas-db --local --file ./drizzle/0001_init.sql

# Run SQL — remote (production)
wrangler d1 execute my-saas-db --remote --command "SELECT count(*) FROM users"
wrangler d1 execute my-saas-db --remote --file ./drizzle/0001_init.sql

# Migrations
wrangler d1 migrations create my-saas-db add_audit_log_table
wrangler d1 migrations apply my-saas-db --local
wrangler d1 migrations apply my-saas-db --remote

# Backup / restore
wrangler d1 export my-saas-db --output backup.sql
wrangler d1 export my-saas-db --remote --output backup.sql
```

**Rule for the agent:** always run migrations against `--local` first to validate, then `--remote` in CI.

### KV

```bash
# Namespaces
wrangler kv namespace create CACHE
wrangler kv namespace create CACHE --preview
wrangler kv namespace list
wrangler kv namespace delete --namespace-id <id>

# Keys
wrangler kv key put --binding CACHE "feature:dark-mode" "true"
wrangler kv key get --binding CACHE "feature:dark-mode"
wrangler kv key list --binding CACHE --prefix "feature:"
wrangler kv key delete --binding CACHE "feature:dark-mode"

# Bulk
wrangler kv bulk put --binding CACHE data.json
wrangler kv bulk delete --binding CACHE keys.json
```

### R2

```bash
# Buckets
wrangler r2 bucket create my-saas-uploads
wrangler r2 bucket list
wrangler r2 bucket delete my-saas-uploads

# Objects
wrangler r2 object put my-saas-uploads/path/file.txt --file ./local.txt
wrangler r2 object get my-saas-uploads/path/file.txt --file ./out.txt
wrangler r2 object list my-saas-uploads
wrangler r2 object delete my-saas-uploads/path/file.txt

# CORS
wrangler r2 bucket cors set my-saas-uploads --rules cors.json
```

### Queues

```bash
wrangler queues create email-jobs
wrangler queues list
wrangler queues delete email-jobs

# Send a test message
wrangler queues producer send email-jobs '{"to":"[email protected]"}'

# Tail consumer
wrangler queues consumer add email-jobs my-worker
```

### Workflows

```bash
wrangler workflows list
wrangler workflows describe <workflow-name>
wrangler workflows trigger <workflow-name> --params '{"userId":"123"}'
wrangler workflows instances list <workflow-name>
wrangler workflows instances describe <workflow-name> <instance-id>
```

### Vectorize

```bash
wrangler vectorize create my-index --dimensions 768 --metric cosine
wrangler vectorize list
wrangler vectorize insert my-index --file vectors.ndjson
wrangler vectorize query my-index --vector "[0.1, 0.2, ...]" --top-k 5
wrangler vectorize delete my-index
```

### Hyperdrive

```bash
wrangler hyperdrive create my-pg --connection-string "postgres://user:pass@host:5432/db"
wrangler hyperdrive list
wrangler hyperdrive update <id> --connection-string "..."
wrangler hyperdrive delete <id>
```

### Workers AI

```bash
# List models
wrangler ai models

# Run a model from CLI for testing
wrangler ai run @cf/meta/llama-3-8b-instruct --prompt "Hello"
```

### Types

```bash
# Generate TypeScript types for all bindings (run after editing wrangler.jsonc)
wrangler types --env-interface CloudflareEnv cloudflare-env.d.ts
```

The agent should **always** regenerate types after changing bindings.

---

## 3. Cloudflare's Official MCP Servers

Cloudflare publishes a catalog of **managed remote MCP servers**. They support OAuth (interactive use) and bearer-token auth (CI/agents). Connect the AI coding agent to whichever ones are relevant to the current task.

All servers support the `streamable-http` transport at `/mcp`. The older `sse` transport is deprecated.

### 3a. The Cloudflare API MCP Server (recommended starting point)

**URL:** `https://mcp.cloudflare.com/mcp`

Covers the **entire Cloudflare API** — over 2,500 endpoints — through just two tools: `search()` and `execute()`. Uses **Code Mode**: the model writes JavaScript that runs in an isolated sandboxed Worker, hitting a local copy of the OpenAPI spec and the Cloudflare client. This collapses what would be a million-token tool list down to ~1,000 tokens of context.

**Use this server when:**
- The agent needs broad access across many Cloudflare products
- The task spans multiple services (e.g., create a Worker + D1 + R2 in one flow)
- Context budget matters

**Connection (any MCP client):**

```json
{
  "mcpServers": {
    "cloudflare-api": {
      "url": "https://mcp.cloudflare.com/mcp"
    }
  }
}
```

Interactive: redirects to Cloudflare OAuth on first connect; the user picks scopes.

CI/automation: pass `Authorization: Bearer <CLOUDFLARE_API_TOKEN>`. Both user and account tokens work. For account tokens, include the **Account Resources: Read** permission so the server can auto-detect account ID.

**Limitation:** API tokens with **Client IP Address Filtering** enabled are not currently supported.

### 3b. Domain-specific MCP servers

For agents working primarily in one Cloudflare area, the curated typed-tool servers are easier to use than the general API server. URLs follow `https://<domain>.mcp.cloudflare.com/mcp`.

| Server | Hostname | Use for |
|---|---|---|
| **Documentation** | `docs.mcp.cloudflare.com` | Querying live Cloudflare docs (better than relying on training data) |
| **Bindings** | `bindings.mcp.cloudflare.com` | Provisioning and inspecting D1, R2, KV bindings during development |
| **Observability** | `observability.mcp.cloudflare.com` | Logs, errors, traces from deployed Workers |
| **Workers Builds** | `builds.mcp.cloudflare.com` | CI build status, build logs |
| **Browser Rendering** | `browser.mcp.cloudflare.com` | Headless browser tasks, screenshots, web scraping |
| **AI Gateway** | `ai-gateway.mcp.cloudflare.com` | LLM call analytics, costs, replay |
| **AutoRAG** | `autorag.mcp.cloudflare.com` | RAG pipeline configuration |
| **Radar** | `radar.mcp.cloudflare.com` | Internet traffic data, threat intel |
| **DNS Analytics** | `dns-analytics.mcp.cloudflare.com` | DNS query analytics, performance |
| **Logpush** | `logpush.mcp.cloudflare.com` | Log export configuration |
| **DEM** | `dem.mcp.cloudflare.com` | Digital Experience Monitoring |
| **GraphQL** | `graphql.mcp.cloudflare.com` | Cloudflare's GraphQL Analytics API |

**Connection (multiple servers, mcp-remote bridge for clients without native remote support):**

```json
{
  "mcpServers": {
    "cloudflare-docs": {
      "command": "npx",
      "args": ["mcp-remote", "https://docs.mcp.cloudflare.com/mcp"]
    },
    "cloudflare-bindings": {
      "command": "npx",
      "args": ["mcp-remote", "https://bindings.mcp.cloudflare.com/mcp"]
    },
    "cloudflare-observability": {
      "command": "npx",
      "args": ["mcp-remote", "https://observability.mcp.cloudflare.com/mcp"]
    }
  }
}
```

### 3c. Cloudflare Skills plugin (for Claude Code, OpenCode, Codex, etc.)

Cloudflare publishes a **Skills plugin** (`cloudflare/skills`) that bundles MCP servers with contextual instructions and slash commands. It works with any agent supporting the Agent Skills standard.

```bash
# Add Wrangler skill specifically
npx skills add https://github.com/cloudflare/skills --skill wrangler
```

The Wrangler skill explicitly tells agents: **"Your knowledge of Wrangler CLI flags, config fields, and subcommands may be outdated. Prefer retrieval over pre-training for any Wrangler task."** This is the right default mindset.

---

## 4. When to Use What

The agent picks based on the operation:

| Task | Best tool |
|---|---|
| Deploy a Worker / Next.js app | `wrangler deploy` (called by `npm run deploy`) |
| Create D1 database, KV namespace, R2 bucket | Wrangler (auto-provision via deploy) or Bindings MCP |
| Run a SQL migration | `wrangler d1 migrations apply` |
| Set a secret | `wrangler secret put` |
| Inspect production logs | `wrangler tail` or Observability MCP |
| Look up Cloudflare docs / how-to | Documentation MCP server |
| Cross-product orchestration (e.g., Workers + DNS + Access in one flow) | Cloudflare API MCP server (Code Mode) |
| One-off resource read (e.g., "list my D1 databases") | Either Wrangler or API MCP — both work |
| Query LLM call analytics | AI Gateway MCP |
| CI/CD scripted deploys | Wrangler in GitHub Actions / Workers Builds |

---

## 5. Setup for Agents Like Devin, Cursor, Claude Code

### Required environment variables for headless agent operation

```bash
CLOUDFLARE_API_TOKEN=<scoped token>
CLOUDFLARE_ACCOUNT_ID=<account id>
```

### Recommended scoped token permissions for a SaaS project

When creating the API token in the Cloudflare dashboard, include:

- Account → Workers Scripts: Edit
- Account → Workers KV Storage: Edit
- Account → Workers R2 Storage: Edit
- Account → D1: Edit
- Account → Queues: Edit
- Account → Workers AI: Read (or Edit if creating fine-tuned models)
- Account → Vectorize: Edit
- Account → Account Settings: Read
- Zone (per zone) → Workers Routes: Edit, DNS: Edit (only if managing custom domains)

Avoid the global API key. Always prefer narrowly scoped tokens.

### Recommended MCP server set for a Next.js + Cloudflare SaaS

Configure these three servers in the agent's MCP client config:

```json
{
  "mcpServers": {
    "cloudflare-api": {
      "url": "https://mcp.cloudflare.com/mcp"
    },
    "cloudflare-docs": {
      "url": "https://docs.mcp.cloudflare.com/mcp"
    },
    "cloudflare-observability": {
      "url": "https://observability.mcp.cloudflare.com/mcp"
    }
  }
}
```

The first handles provisioning, the second prevents the agent from using stale training data on Cloudflare APIs, the third gives logs/error visibility post-deploy.

### Recommended local setup checklist for the agent

1. `npm i -D wrangler@latest @opennextjs/cloudflare@latest`
2. Create or verify `wrangler.jsonc` with required bindings
3. Confirm `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` are set
4. Run `wrangler whoami` to verify auth
5. Run `wrangler types` to generate binding types
6. Add `.dev.vars` to `.gitignore` if using local secrets
7. Add `.open-next/` to `.gitignore`

---

## 6. Common Operations: Recipes

### Recipe: Bootstrap a new D1 database with schema

```bash
# 1. Create the database (writes ID to wrangler.jsonc if binding is configured)
wrangler d1 create my-saas-db

# 2. Apply schema (Drizzle output, or hand-written SQL)
wrangler d1 execute my-saas-db --local --file ./drizzle/0001_init.sql
wrangler d1 execute my-saas-db --remote --file ./drizzle/0001_init.sql

# 3. Verify
wrangler d1 execute my-saas-db --remote --command "SELECT name FROM sqlite_master WHERE type='table'"
```

### Recipe: Set up R2 bucket with CORS for browser uploads

```bash
wrangler r2 bucket create my-saas-uploads

cat > cors.json <<'EOF'
[
  {
    "AllowedOrigins": ["https://my-saas.com"],
    "AllowedMethods": ["GET", "PUT", "HEAD"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]
EOF

wrangler r2 bucket cors set my-saas-uploads --rules cors.json
```

### Recipe: Add a queue with a consumer

In `wrangler.jsonc`:

```jsonc
"queues": {
  "producers": [{ "binding": "EMAIL_QUEUE", "queue": "email-jobs" }],
  "consumers": [
    {
      "queue": "email-jobs",
      "max_batch_size": 10,
      "max_batch_timeout": 30,
      "max_retries": 3,
      "dead_letter_queue": "email-jobs-dlq"
    }
  ]
}
```

```bash
wrangler queues create email-jobs
wrangler queues create email-jobs-dlq
wrangler deploy
```

### Recipe: Promote staging to production after smoke tests

```bash
wrangler deploy --env staging
# run e2e tests against staging URL
wrangler deploy --env production
```

### Recipe: Roll back a bad deploy

```bash
wrangler versions list
wrangler rollback <previous-version-id>
```

### Recipe: Investigate a production incident

```bash
# Stream errors only
wrangler tail --env production --status error --format json

# Inspect D1 state
wrangler d1 execute my-saas-db --remote --command "SELECT * FROM jobs WHERE status='failed' ORDER BY created_at DESC LIMIT 20"
```

---

## 7. Things the Agent Should NOT Do

- **Do not run `wrangler login` in non-interactive contexts.** Use `CLOUDFLARE_API_TOKEN`.
- **Do not put secrets in `wrangler.jsonc` or `vars`.** Use `wrangler secret put` for production, `.dev.vars` for local.
- **Do not use the global API key.** Always create a scoped token.
- **Do not run `wrangler d1 execute --remote` for destructive operations** (DROP, DELETE without WHERE) without explicit confirmation. Always test on `--local` first.
- **Do not commit `.dev.vars`, `.open-next/`, or `wrangler.toml.bak` to git.**
- **Do not assume training-data knowledge of Wrangler flags** is current. Wrangler's CLI surface evolves frequently. When in doubt, run `wrangler <command> --help` or query the Documentation MCP.
- **Do not use the deprecated `sse` MCP transport.** Use `streamable-http` (the `/mcp` endpoint).
- **Do not use `wrangler.toml` for new projects.** Use `wrangler.jsonc`.
- **Do not skip `wrangler types` after binding changes.** Type drift causes hard-to-debug runtime errors.
- **Do not call the Cloudflare REST API directly with raw `fetch`** when Wrangler or an MCP server can handle it.

---

## 8. Decision Heuristic for the Agent

When the user asks for a Cloudflare operation, walk this list:

1. Is this a **deploy/build/dev** operation? → `wrangler` command in the project.
2. Is this **resource provisioning** (create DB, bucket, namespace)? → Add binding to `wrangler.jsonc`, run `wrangler deploy` to auto-provision; or use the matching `wrangler <product> create` command.
3. Is this **operational** (logs, debug, state inspection)? → `wrangler tail` for logs; `wrangler d1 execute --remote` for DB state; or the Observability MCP server.
4. Is this **cross-product** or **API-level** (e.g., create a Workers + DNS + Access setup)? → Cloudflare API MCP server with `search()` then `execute()`.
5. Is this **a question about how Cloudflare works**? → Documentation MCP server, never training data.
6. Is this something **none of the above** can do? → Cloudflare REST API directly, but wrap it in a script and document why Wrangler/MCP wasn't sufficient.

---

*Last updated: April 2026. Verify command syntax with `wrangler <command> --help` or the Documentation MCP server before committing to any tier-sensitive or destructive operation.*
