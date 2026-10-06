## How to run this project (read this first)

These instructions govern initial builds, maintenance, deployment, and teardown.

- Initial builds: complete Phase 1 and Phase 2 before writing application or launch code. Read-only inspection and writing the plan are allowed before these gates pass.
- Maintenance: reuse the approved plan. Ask for explicit approval only when a requested change materially changes scope, architecture, data placement, or design. Re-check Phase 2 before deployment or changes to deployment targets or required secrets; routine local fixes do not require production credentials.
- Documentation-only reviews and edits: build and deployment gates do not apply.
- Deployment and teardown require user authorization. Authorization to edit or push repository files does not by itself authorize triggering a deployment. Before pushing, check whether the push will trigger deployment; if so, obtain deployment authorization or use an established mechanism to skip that deployment.
- Preserve approval and verification evidence in `docs/plan.md`, without secret values. Reuse existing answers and approvals rather than restarting completed phases.

### Phase 1 — Questions and planning

1. Read existing project documentation and the approved plan first. Ask unanswered questions in batches covering the app’s purpose, users, core features, data, files, background jobs, scheduled jobs, and design. Required decisions are those that affect scope, architecture, security, data placement, launch URL, or design. Resolve these with the user; propose and document reasonable assumptions for minor implementation details instead of asking indefinitely.
2. Write `docs/plan.md` covering features, data model, API endpoints, pages, background and scheduled jobs, partition keys (which entities live in Durable Objects and what each object is keyed by), required Cloudflare services, deploy targets, final URL, assumptions, and a launch checklist. Phase 1 is complete when required decisions are resolved, a design is supplied or the user explicitly authorizes an agent-created design, and the user explicitly approves the plan. Record that approval in the plan before implementation.
3. Design: suggest the user ask their Muse agent to explore design directions using the Figma connector, then hand you the resulting design or Figma link to implement. Do not invent a full visual design unprompted. Implement the design you are given, or ask for one. An explicit request to create the design authorizes doing so; record that decision in the plan.

### Phase 2 — Launch credentials check

Secrets live in GitHub Environments, not in the terminal. There is one GitHub Environment per deploy target (typically just `production`; add `staging` only if the plan calls for it). For initial builds, verify both of these before writing application or launch code. For maintenance, verify them before deployment or changes to deployment targets or required secrets:

- The deploy environment exists in the repo (Settings, then Environments) and holds every secret the deploy workflow needs: at minimum CLOUDFLARE\_API\_TOKEN, BETTER\_AUTH\_SECRET, and BETTER\_AUTH\_URL. Check with `gh secret list --env <name>` (secret names only, never print values).
- The `gh` CLI is authenticated locally (`gh auth status` succeeds) so the agent can read secret names and trigger workflows.

The deploy workflow passes secrets to wrangler as environment variables (for example `CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}`). For one-off interactive wrangler commands on your own machine, export the token by hand in that terminal session only — never persist it in shell rc files, and never commit it anywhere.

If the environment or any secret is missing, stop and ask the user to create it in the repo's Settings, then Environments page: name the environment, list each missing secret with where its value comes from (CLOUDFLARE\_API\_TOKEN from a new custom token per the permission list below; BETTER\_AUTH\_SECRET from `openssl rand -base64 32`; BETTER\_AUTH\_URL from the plan's final URL), and show the permission list below for the Cloudflare token. Wait for the user to confirm the secrets are in place, then re-check with `gh secret list --env <name>` before proceeding. Do not attempt a launch without all secrets present.

When the Cloudflare token is missing or insufficient, show the user this exact permission list so they can create a proper custom token (Cloudflare dashboard, then My Profile, then API Tokens, then Create Custom Token):

Account scope:

- Workers: Admin (this is the current name in the Cloudflare dashboard; it replaces the old "Workers Scripts: Edit". It must be granted at product scope, which covers creating new Workers)
- D1: Edit
- Workers R2 Storage: Edit
- Queues: Edit
- Account Settings: Read

Zone scope (only needed if the app will use a custom domain on a Cloudflare-managed zone; skip entirely when using the workers.dev subdomain):

- DNS: Edit
- Workers Routes: Edit

If the approved plan uses KV, also grant Workers KV Storage: Edit at account scope.

The token must include the account under Account Resources. The user can sanity-check a pasted token by running `export CLOUDFLARE_API_TOKEN=[redacted] # paste once, session only` followed by `wrangler d1 list` — it should succeed.

### Documentation lookup — verify library APIs

Libraries change. Before writing code that touches any library API, look up its current documentation with Context7 instead of relying on memory. This applies to everything in the locked stack: Hono, Drizzle, drizzle-zod, Better Auth, React, Vite, Tailwind, and the Cloudflare Workers, D1, Durable Objects, R2, Queues, Workflows, and wrangler APIs.

1. Check whether Context7 library resolution and documentation tools are available. Tool names may vary by host; use the tools’ descriptions to identify the equivalent capabilities.
2. If available, resolve the library ID, then retrieve documentation for the relevant topic and installed or planned version. Do this before writing Hono routes, Drizzle queries and migrations, Better Auth configuration, and wrangler commands. Reuse documentation already retrieved when its version and topic still apply.
3. If Context7 is unavailable, tell the user and consult official documentation instead. For Claude Code, setup is `claude mcp add --transport http context7 https://mcp.context7.com/mcp`; for other hosts, configure the HTTP MCP server at `https://mcp.context7.com/mcp` using that host’s supported setup. Do not run another host’s setup command automatically. Continue with official documentation and flag any API behavior that remains unverified.
4. When verified documentation disagrees with assumptions, the documentation wins.

### Phase 3 — Build

The stack (locked, do not substitute). Workers, D1, Static Assets, and the listed libraries are the application baseline. Durable Objects, R2, Queues, Workflows, Cron Triggers, and KV are available services: add bindings and provision resources only when the approved plan requires them.

- Cloudflare: Workers, D1, Durable Objects (SQLite storage), R2, Queues, Workflows, Cron Triggers, Static Assets, KV (optional — add a binding only when the plan justifies it under the data placement rule).
- Libraries: Hono (API), Drizzle (database access), Zod via drizzle-zod (validation), Better Auth (self-hosted logins).
- Secrets: GitHub Environments in CI; a gitignored `.dev.vars` file for local `wrangler dev`. No committed secret files, ever.
- Frontend: React plus Vite plus Tailwind CSS, built to static files and served from the same Worker as the API.

Conventions:

- One Worker serves everything: the Hono API under /api/v1 as REST JSON, Better Auth at /api/auth/\*, and the frontend static assets for everything else with SPA fallback.
- The Drizzle schema in src/db/schema.ts is the single source of truth for all tables, including Better Auth's tables. Derive Zod schemas from it with drizzle-zod. Never hand-write duplicate database field schemas. Derive database-backed request fields from these schemas and extend or compose them as needed. Write separate Zod schemas for inputs with no database equivalent, such as pagination, search parameters, and action payloads.
- Data placement (the scaling rule): global and relational data lives in D1 — accounts, Better Auth tables, settings, anything queried across entities. High-write partitioned state lives in Durable Objects with SQLite storage, one object per partition key (user, workspace, room, document) chosen at plan time. Each object has its own SQLite database and throughput, so writes scale by adding partitions. Reads scale via D1 read replication. Never funnel high-volume entity writes through the single D1 database. KV is allowed only for read-heavy, low-write, eventually-consistent data (feature flags, public config, cached computed values, short-lived tokens with TTL) — never for relational data, high-write state, or files.
- Validate every application API input, including path parameters, query parameters, and request bodies. Under `/api/v1`, errors use the envelope `{ error: { code, message } }` and lists paginate with `?page=` and `?per_page=`. Document pagination defaults, limits, and response metadata. Better Auth owns `/api/auth/*`; preserve its documented request, response, and error contracts.
- Better Auth runs inside the Worker with its data in D1. Its secrets (BETTER\_AUTH\_SECRET, BETTER\_AUTH\_URL) come from the deploy environment's GitHub secrets and are synced to Worker secrets at deploy time by the deploy workflow.
- Files go in R2. Use presigned URLs so browsers upload directly instead of proxying large files through the Worker.
- Background work goes on the queue. Multi-step durable work (retries, sleeps, sequential steps) goes on Workflows. Recurring work goes on cron triggers. Never build an always-on scheduler process.
- Local development is plain `wrangler dev`, which gives a full local stack: local D1 as a SQLite file, emulated R2, and locally persisted Durable Objects. App secrets for local runs come from a gitignored `.dev.vars` file (commit only a `.dev.vars.example` template with empty values). `wrangler dev` runs fully local and never needs the Cloudflare token; it only goes out to Cloudflare if the agent passes --remote, which it should not do without asking first.
- Keep write volume low: rows written is the dominant cost line at scale, so never rewrite rows needlessly and batch related writes.
- Keep the JSON API stable and documented. A future iOS app will consume this same API.

### Phase 4 — Launch

Prerequisites: the deploy environment holds all secrets (Phase 2 verified). All launch steps run in GitHub Actions via the deploy workflow, which exposes the environment's secrets to wrangler as environment variables. Never run a launch from a laptop terminal with a pasted token unless the user explicitly asks for a manual deploy.

1. Install dependencies and build the frontend.
2. On first launch, create the D1 database and any R2 buckets, queues, or KV namespaces required by the approved plan. On later deploys, reuse existing resources. Persist resource names and required IDs in `wrangler.jsonc` through a reviewed repository change, or in documented durable per-environment configuration that CI uses to generate its deploy configuration. Never rely on edits to an ephemeral CI checkout surviving the run. Declare required Durable Object classes (`new_sqlite_classes`) and Workflows in `wrangler.jsonc`; they need no separate creation step. Provisioning must be repeatable: detect existing resources, fail clearly on ambiguous matches, and never delete or replace data-bearing resources to make a deploy succeed.
3. Apply the Drizzle migrations to the remote D1 database.
4. Sync the environment's secrets (BETTER\_AUTH\_SECRET, BETTER\_AUTH\_URL set to the final URL, plus any app secrets) to Worker secrets.
5. Run `wrangler deploy`.
6. Verify on the live URL: /healthz responds, signup and login work, and one record can be created end to end through the API.

Encode these steps in `.github/workflows/deploy.yml` (triggered on pushes to main, or manually, as specified in the approved plan) so a future agent can relaunch without rethinking them. The workflow must use `environment: <name>` so it reads that environment’s secrets, and must fail fast with a clear message if any required secret is missing. Serialize deployments per environment to avoid competing provisioning or migration runs. Document the deployment trigger and any supported way to push without deploying in `docs/plan.md`.

### Phase 5 — Teardown (unlaunch)

Only do this when the user explicitly asks to take the project down. Teardown is destructive and permanent: deleted data cannot be recovered. Always get the user's explicit confirmation before running any of it, and never tear down as part of a launch, redeploy, or routine cleanup. Run automated teardown steps through the teardown workflow, which exposes the environment’s secrets to wrangler. Clearly identify any dashboard-only cleanup as a manual user step. Only run teardown commands from a laptop terminal with a session-exported token if the user explicitly asks.

1. Optional but recommended: back up data before deleting anything. Export the D1 database: `wrangler d1 export <database-name> --output=backup-<name>-<date>.sqlite`. Download any R2 objects worth keeping. Ask the user if they want backups; if they skip it, proceed without.
2. If the app uses a custom domain, remove the Worker's route or custom domain first: on a Cloudflare-managed zone, delete the DNS record and the Workers route. The workers.dev subdomain is handled in the next step.
3. Delete the Worker: `wrangler delete <worker-name>`. The workers.dev subdomain, its scheduled cron triggers, and its routes go away with it. If the app used Durable Objects, remove their namespace data via the Cloudflare dashboard or API as well.
4. If provisioned, delete the queue: `wrangler queues delete <queue-name>`.
5. Delete the D1 database: `wrangler d1 delete <database-name>`. This permanently destroys all app data.
6. If provisioned, empty the R2 bucket by deleting all of its objects, then delete the bucket: `wrangler r2 bucket delete <bucket-name>`.
7. If the app uses KV, delete each namespace: `wrangler kv namespace delete <namespace-id>`.
8. In the repo's Settings, then Environments page, delete the app's secrets (or the whole environment) so the deploy credentials are gone too. Optionally revoke the Cloudflare API token as well.
9. Verify everything is gone: `wrangler d1 list`, `wrangler r2 bucket list`, and the Workers dashboard should show nothing left for this app. Hitting the old URL should respond with a 404 from Cloudflare, not the application.

Encode these steps in `.github/workflows/teardown.yml` (manual dispatch only) so a future agent can tear down without rethinking them. The workflow must require a typed confirmation input before deleting anything, and must offer the backup step first.

## Rules

- KV only with plan-time justification. No committed secret files (`.dev.vars` stays gitignored; only `.dev.vars.example` is committed). No code generator CLI. Single tenant.
- Never tear down without the user's explicit approval.
- TypeScript strict. Keep the code plain and simple, no clever abstractions.
- Never commit secrets. Never print tokens.
