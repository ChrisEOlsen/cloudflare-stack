## How to run this project (read this first)

You are the build agent. Do not write any code until you have completed Phase 1 and Phase 2 below. The user wants a rigorous planning stage before any implementation.

### Phase 1 — Questions and planning

1. Ask the user every question you need answered to build this project: what the app does, who uses it, the core features, what data it stores, what files it handles, what background jobs and scheduled jobs it needs, and what the design should look like. Keep asking until nothing material is unanswered.
2. Write up a plan covering features, data model, API endpoints, pages, background and scheduled jobs, and a launch checklist. Get the user's explicit approval on the plan before building anything.
3. Design: suggest the user ask their Muse agent to explore design directions using the Figma connector, then hand you the resulting design or Figma link to implement. Do not invent a full visual design unprompted. Implement the design you are given, or ask for one.

### Phase 2 — Launch credentials check

Before writing any launch code, verify both of these exist in the environment:

- CLOUDFLARE_API_TOKEN — deploys and manages everything on Cloudflare.
- DOPPLER_TOKEN — a Doppler service token for this app's project, used to inject secrets.

If either is missing, stop and tell the user exactly which one is missing and how to provide it. Do not attempt a launch without both.

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

The token must include the account under Account Resources. The user can sanity-check a token by running `wrangler d1 list` — it should succeed.

### Phase 3 — Build

The stack (locked, do not substitute):

- Cloudflare: Workers, D1, R2, Queues, Cron Triggers, Static Assets. No KV. Do not add a KV binding.
- Libraries: Hono (API), Drizzle (database access), Zod via drizzle-zod (validation), Better Auth (self-hosted logins).
- Secrets: Doppler only. No .env files anywhere, ever.
- Frontend: React plus Vite plus Tailwind CSS, built to static files and served from the same Worker as the API.

Conventions:

- One Worker serves everything: the Hono API under /api/v1 as REST JSON, Better Auth at /api/auth/*, and the frontend static assets for everything else with SPA fallback.
- The Drizzle schema in src/db/schema.ts is the single source of truth for all tables, including Better Auth's tables. Derive Zod schemas from it with drizzle-zod. Never hand-write duplicate schemas.
- Validate every API input. Errors use the envelope { error: { code, message } }. Paginate lists with ?page= and ?per_page=.
- Better Auth runs inside the Worker with its data in D1. Its secrets (BETTER_AUTH_SECRET, BETTER_AUTH_URL) come from Doppler and are synced to Worker secrets at deploy time.
- Files go in R2. Use presigned URLs so browsers upload directly instead of proxying large files through the Worker.
- Background work goes on the queue. Recurring work goes on cron triggers. Never build an always-on scheduler process.
- Local development is `doppler run -- wrangler dev`, which gives a full local stack: local D1 as a SQLite file and emulated R2.
- Keep the JSON API stable and documented. A future iOS app will consume this same API.

### Phase 4 — Launch

Prerequisites: both tokens from Phase 2 are verified.

1. Install dependencies and build the frontend.
2. Create the D1 database, the R2 bucket, and the queue with wrangler. Record their IDs in wrangler.jsonc.
3. Apply the Drizzle migrations to the remote D1 database.
4. Set the secrets in Doppler (generate BETTER_AUTH_SECRET, set BETTER_AUTH_URL to the final URL), then sync them to Worker secrets.
5. Run wrangler deploy.
6. Verify on the live URL: /healthz responds, signup and login work, and one record can be created end to end through the API.

Encode these steps in scripts/launch.sh so a future agent can relaunch without rethinking them.

## Rules

- No KV. No .env files. No code generator CLI. Single tenant.
- TypeScript strict. Keep the code plain and simple, no clever abstractions.
- Never commit secrets. Never print tokens.

