## How to run this project (read this first)

You are the build agent. Do not write any code until you have completed Phase 1 and Phase 2 below. The user wants a rigorous planning stage before any implementation.

### Phase 1 — Questions and planning

1. Ask the user every question you need answered to build this project: what the app does, who uses it, the core features, what data it stores, what files it handles, what background jobs and scheduled jobs it needs, and what the design should look like. Keep asking until nothing material is unanswered.
2. Write up a plan covering features, data model, API endpoints, pages, background and scheduled jobs, partition keys (which entities live in Durable Objects and what each object is keyed by), and a launch checklist. Get the user's explicit approval on the plan before building anything.
3. Design: suggest the user ask their Muse agent to explore design directions using the Figma connector, then hand you the resulting design or Figma link to implement. Do not invent a full visual design unprompted. Implement the design you are given, or ask for one.

### Phase 2 — Launch credentials check

Secrets live in GitHub Environments, not in the terminal. There is one GitHub Environment per deploy target (typically just `production`; add `staging` only if the plan calls for it). Verify both of these before writing any launch code:

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

### Context7 — look up current API docs

Libraries change. Before writing code that touches any library API, look up its current documentation with Context7 instead of relying on memory. This applies to everything in the locked stack: Hono, Drizzle, drizzle-zod, Better Auth, React, Vite, Tailwind, and the Cloudflare Workers, D1, Durable Objects, R2, Queues, Workflows, and wrangler APIs.

1. Check whether Context7 is available in your environment (MCP tools named `resolve-library-id` and `get-library-docs`).
2. If it is not available, tell the user it is missing and how to add it: `claude mcp add --transport http context7 https://mcp.context7.com/mcp`. Then continue, flagging anything you could not verify.
3. Workflow: call `resolve-library-id` with the library name to get its Context7 ID, then call `get-library-docs` with that ID and the topic you need. Do this before writing Hono routes, Drizzle queries and migrations, Better Auth configuration, and wrangler commands.
4. When the docs disagree with your assumptions, the docs win.

### Phase 3 — Build

The stack (locked, do not substitute):

- Cloudflare: Workers, D1, Durable Objects (SQLite storage), R2, Queues, Workflows, Cron Triggers, Static Assets, KV (optional — add a binding only when the plan justifies it under the data placement rule).
- Libraries: Hono (API), Drizzle (database access), Zod via drizzle-zod (validation), Better Auth (self-hosted logins).
- Secrets: GitHub Environments in CI; a gitignored `.dev.vars` file for local `wrangler dev`. No committed secret files, ever.
- Frontend: React plus Vite plus Tailwind CSS, built to static files and served from the same Worker as the API.

Conventions:

- One Worker serves everything: the Hono API under /api/v1 as REST JSON, Better Auth at /api/auth/\*, and the frontend static assets for everything else with SPA fallback.
- The Drizzle schema in src/db/schema.ts is the single source of truth for all tables, including Better Auth's tables. Derive Zod schemas from it with drizzle-zod. Never hand-write duplicate schemas.
- Data placement (the scaling rule): global and relational data lives in D1 — accounts, Better Auth tables, settings, anything queried across entities. High-write partitioned state lives in Durable Objects with SQLite storage, one object per partition key (user, workspace, room, document) chosen at plan time. Each object has its own SQLite database and throughput, so writes scale by adding partitions. Reads scale via D1 read replication. Never funnel high-volume entity writes through the single D1 database. KV is allowed only for read-heavy, low-write, eventually-consistent data (feature flags, public config, cached computed values, short-lived tokens with TTL) — never for relational data, high-write state, or files.
- Validate every API input. Errors use the envelope { error: { code, message } }. Paginate lists with ?page= and ?per\_page=.
- Better Auth runs inside the Worker with its data in D1. Its secrets (BETTER\_AUTH\_SECRET, BETTER\_AUTH\_URL) come from the deploy environment's GitHub secrets and are synced to Worker secrets at deploy time by the deploy workflow.
- Files go in R2. Use presigned URLs so browsers upload directly instead of proxying large files through the Worker.
- Background work goes on the queue. Multi-step durable work (retries, sleeps, sequential steps) goes on Workflows. Recurring work goes on cron triggers. Never build an always-on scheduler process.
- Local development is plain `wrangler dev`, which gives a full local stack: local D1 as a SQLite file, emulated R2, and locally persisted Durable Objects. App secrets for local runs come from a gitignored `.dev.vars` file (commit only a `.dev.vars.example` template with empty values). `wrangler dev` runs fully local and never needs the Cloudflare token; it only goes out to Cloudflare if the agent passes --remote, which it should not do without asking first.
- Keep write volume low: rows written is the dominant cost line at scale, so never rewrite rows needlessly and batch related writes.
- Keep the JSON API stable and documented. A future iOS app will consume this same API.

### Phase 4 — Launch

Prerequisites: the deploy environment holds all secrets (Phase 2 verified). All launch steps run in GitHub Actions via the deploy workflow, which exposes the environment's secrets to wrangler as environment variables. Never run a launch from a laptop terminal with a pasted token unless the user explicitly asks for a manual deploy.

1. Install dependencies and build the frontend.
2. Create the D1 database, the R2 bucket, and the queue. Record their IDs in wrangler.jsonc. Declare Durable Object classes (new_sqlite_classes) and Workflows in wrangler.jsonc — they need no separate creation step. If the plan uses KV, create the namespace and record its ID as well.
3. Apply the Drizzle migrations to the remote D1 database.
4. Sync the environment's secrets (BETTER\_AUTH\_SECRET, BETTER\_AUTH\_URL set to the final URL, plus any app secrets) to Worker secrets.
5. Run `wrangler deploy`.
6. Verify on the live URL: /healthz responds, signup and login work, and one record can be created end to end through the API.

Encode these steps in `.github/workflows/deploy.yml` (triggered on pushes to main, or manually) so a future agent can relaunch without rethinking them. The workflow must use `environment: <name>` so it reads that environment's secrets, and must fail fast with a clear message if any required secret is missing.

### Phase 5 — Teardown (unlaunch)

Only do this when the user explicitly asks to take the project down. Teardown is destructive and permanent: deleted data cannot be recovered. Always get the user's explicit confirmation before running any of it, and never tear down as part of a launch, redeploy, or routine cleanup. Run every step here through the teardown workflow, which exposes the environment's secrets to wrangler. Only run teardown commands from a laptop terminal with a session-exported token if the user explicitly asks.

1. Optional but recommended: back up data before deleting anything. Export the D1 database: `wrangler d1 export <database-name> --output=backup-<name>-<date>.sqlite`. Download any R2 objects worth keeping. Ask the user if they want backups; if they skip it, proceed without.
2. If the app uses a custom domain, remove the Worker's route or custom domain first: on a Cloudflare-managed zone, delete the DNS record and the Workers route. The workers.dev subdomain is handled in the next step.
3. Delete the Worker: `wrangler delete <worker-name>`. The workers.dev subdomain, its scheduled cron triggers, and its routes go away with it. If the app used Durable Objects, remove their namespace data via the Cloudflare dashboard or API as well.
4. Delete the queue: `wrangler queues delete <queue-name>`.
5. Delete the D1 database: `wrangler d1 delete <database-name>`. This permanently destroys all app data.
6. Empty the R2 bucket by deleting all of its objects, then delete the bucket: `wrangler r2 bucket delete <bucket-name>`.
7. If the app uses KV, delete each namespace: `wrangler kv namespace delete <namespace-id>`.
8. In the repo's Settings, then Environments page, delete the app's secrets (or the whole environment) so the deploy credentials are gone too. Optionally revoke the Cloudflare API token as well.
9. Verify everything is gone: `wrangler d1 list`, `wrangler r2 bucket list`, and the Workers dashboard should show nothing left for this app. Hitting the old URL should respond with a 404 from Cloudflare, not the application.

Encode these steps in `.github/workflows/teardown.yml` (manual dispatch only) so a future agent can tear down without rethinking them. The workflow must require a typed confirmation input before deleting anything, and must offer the backup step first.

## Rules

- KV only with plan-time justification. No committed secret files (`.dev.vars` stays gitignored; only `.dev.vars.example` is committed). No code generator CLI. Single tenant.
- Never tear down without the user's explicit approval.
- TypeScript strict. Keep the code plain and simple, no clever abstractions.
- Never commit secrets. Never print tokens.
