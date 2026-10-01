# AGENTS.md

Drop this file in an empty project folder and open your AI coding agent here. It tells the agent exactly how to run the project: plan first, ask questions, verify launch credentials, then build, then launch.

## How to run this project (read this first)

You are the build agent. Do not write any code until you have completed Phase 1 and Phase 2 below. The user wants a rigorous planning stage before any implementation.

### Phase 1 — Questions and planning

1. Ask the user every question you need answered to build this project: what the app does, who uses it, the core features, what data it stores, what files it handles, what background jobs and scheduled jobs it needs, and what the design should look like. Keep asking until nothing material is unanswered.
2. Write up a plan covering features, data model, API endpoints, pages, background and scheduled jobs, and a launch checklist. Get the user's explicit approval on the plan before building anything.
3. Design: suggest the user ask their Muse agent to explore design directions using the Figma connector, then hand you the resulting design or Figma link to implement. Do not invent a full visual design unprompted. Implement the design you are given, or ask for one.

### Phase 2 — Launch credentials check

The Cloudflare API token lives in Doppler dev secrets, not in the terminal. Verify both of these before writing any launch code:

- DOPPLER_TOKEN exists in the environment. This one cannot come from Doppler itself, so it must be set in the terminal.
- CF_API_TOKEN exists in the Doppler dev config for this app's project. Check with `doppler secrets` (secret names only, never print values).

Wrangler expects the token in the CLOUDFLARE_API_TOKEN environment variable, while Doppler stores it as CF_API_TOKEN. Bridge this in scripts by putting this line at the top, before any wrangler command:
`export CLOUDFLARE_API_TOKEN="$(doppler secrets get CF_API_TOKEN --plain)"`
For one-off interactive commands, prefix them the same way: `CLOUDFLARE_API_TOKEN="$(doppler secrets get CF_API_TOKEN --plain)" wrangler ...`.
Never put the token in the terminal environment permanently; always fetch it from Doppler at runtime.

If DOPPLER_TOKEN is missing, stop and tell the user to set it in the terminal. If CF_API_TOKEN is missing from the Doppler dev config, stop and tell the user to add it there, showing the permission list below. Do not attempt a launch without both.

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

The token must include the account under Account Resources. The user can sanity-check a token by running `CLOUDFLARE_API_TOKEN="$(doppler secrets get CF_API_TOKEN --plain)" wrangler d1 list` — it should succeed.

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
- Local development is `doppler run -- wrangler dev`, which gives a full local stack: local D1 as a SQLite file and emulated R2. `wrangler dev` runs fully local and never needs the Cloudflare token; it only goes out to Cloudflare if the agent passes --remote, which it should not do without asking first.
- Keep the JSON API stable and documented. A future iOS app will consume this same API.

### Phase 4 — Launch

Prerequisites: DOPPLER_TOKEN is set in the terminal and CF_API_TOKEN is in the Doppler dev config (Phase 2 verified). Run every wrangler command under the CF_API_TOKEN export from Phase 2, not as `doppler run -- wrangler ...` (which would not give wrangler the name it expects).

1. Install dependencies and build the frontend.
2. Create the D1 database, the R2 bucket, and the queue. Record their IDs in wrangler.jsonc.
3. Apply the Drizzle migrations to the remote D1 database.
4. Set the secrets in Doppler (generate BETTER_AUTH_SECRET, set BETTER_AUTH_URL to the final URL), then sync them to Worker secrets.
5. Run `wrangler deploy` with the token exported.
6. Verify on the live URL: /healthz responds, signup and login work, and one record can be created end to end through the API.

Encode these steps in scripts/launch.sh so a future agent can relaunch without rethinking them. The script must export CLOUDFLARE_API_TOKEN from CF_API_TOKEN at the top, and must fail with a clear message if CF_API_TOKEN is missing from Doppler.

### Phase 5 — Teardown (unlaunch)

Only do this when the user explicitly asks to take the project down. Teardown is destructive and permanent: deleted data cannot be recovered. Always get the user's explicit confirmation before running any of it, and never tear down as part of a launch, redeploy, or routine cleanup. Run every wrangler command here under the CF_API_TOKEN export from Phase 2.

1. Optional but recommended: back up data before deleting anything. Export the D1 database: `CLOUDFLARE_API_TOKEN="$(doppler secrets get CF_API_TOKEN --plain)" wrangler d1 export <database-name> --output=backup-<name>-<date>.sqlite`. Download any R2 objects worth keeping. Ask the user if they want backups; if they skip it, proceed without.
2. If the app uses a custom domain, remove the Worker's route or custom domain first: on a Cloudflare-managed zone, delete the DNS record and the Workers route. The workers.dev subdomain is handled in the next step.
3. Delete the Worker: `wrangler delete <worker-name>`. The workers.dev subdomain, its scheduled cron triggers, and its routes go away with it.
4. Delete the queue: `wrangler queues delete <queue-name>`.
5. Delete the D1 database: `wrangler d1 delete <database-name>`. This permanently destroys all app data.
6. Empty the R2 bucket by deleting all of its objects, then delete the bucket: `wrangler r2 bucket delete <bucket-name>`.
7. In the Doppler dashboard, archive or delete the app's Doppler project so its secrets are gone too. Revoke the service token used for this deployment.
8. Verify everything is gone: `wrangler d1 list`, `wrangler r2 bucket list`, and the Workers dashboard should show nothing left for this app. Hitting the old URL should respond with a 404 from Cloudflare, not the application.

Encode these steps in scripts/teardown.sh so a future agent can tear down without rethinking them. The script must export CLOUDFLARE_API_TOKEN from CF_API_TOKEN at the top, ask for confirmation before deleting anything, and offer the backup step first.

## Rules

- No KV. No .env files. No code generator CLI. Single tenant.
- Never tear down without the user's explicit approval.
- TypeScript strict. Keep the code plain and simple, no clever abstractions.
- Never commit secrets. Never print tokens.

