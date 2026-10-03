# How this system works — a system design walkthrough

This file explains the architecture in `AGENTS.md`: one Cloudflare Worker
serving an API plus a frontend, backed by two kinds of database, object
storage, and three kinds of background execution. Read it as a lesson in
*why the pieces are arranged this way*, not just what they are.

## 1. The big picture

Everything lives inside Cloudflare's network. There are no servers to
manage, no VPCs, no connection pools to tune. The whole backend is one
Worker plus the services it is bound to:

```
                        Cloudflare edge (300+ locations worldwide)
  ┌──────────┐         ┌─────────────────────────────────────────────────┐
  │ Browser /│  HTTPS  │  ┌─────────────── Worker ───────────────────┐    │
  │ iOS app  │ ──────▶ │  │  /api/v1 ......... Hono API (REST JSON)  │    │
  └──────────┘         │  │  /api/auth/* ..... Better Auth (login)   │    │
                       │  │  /* .............. static SPA fallback   │    │
                       │  └──────┬───┬───┬───┬───┬───────────────────┘    │
                       │         │   │   │   │   │                        │
                       │    ┌────┘   │   │   │   └────┐                   │
                       │    ▼        ▼   ▼   ▼        ▼                   │
                       │   ┌───┐  ┌────┐ ┌──┐ ┌─────┐ ┌──────────┐        │
                       │   │ D1│  │Dur-│ │R2│ │Queue│ │Workflow  │        │
                       │   │   │  │able│ │  │ │     │ │+ Cron    │        │
                       │   └───┘  │Objs│ └──┘ └─────┘ └──────────┘        │
                       │          └────┘                                  │
                       └─────────────────────────────────────────────────┘
```

Two facts explain most of the design:

1. **The Worker is stateless and infinitely scalable.** Every request can
   run on any machine. Anything that must be remembered has to live in one
   of the boxes below it.
2. **Each box below it scales differently.** The art of this stack is
   putting each kind of data in the box whose scaling matches it.

## 2. Piece by piece: what each part does and why it exists

| Piece | Job | Why it, and not something else |
|---|---|---|
| Worker + Hono | Runs the API: routing, validation, auth checks, then one or two storage calls per request | Scales to zero and up with no config; you never think about it |
| D1 | The one shared relational database (SQLite engine, Cloudflare-managed storage) | Holds everything queried *across* entities: users, logins, settings, billing |
| Durable Objects | One tiny stateful object per partition key (user, workspace, room), each with its own private SQLite database | Holds high-write state so writes spread across thousands of independent databases instead of queuing on one |
| R2 | File/blob storage | Files never touch the database; browsers upload straight to R2 via presigned URLs |
| Queue | Fire-and-forget background tasks | Absorbs bursts: the API answers fast and the heavy work drains at its own pace |
| Workflow | Multi-step durable jobs (retries, sleeps, sequences) | The thing a plain queue cannot express: "try, wait an hour, try again, then notify" |
| Cron Triggers | Scheduled entry points ("every hour, wake up") | Recurring work with no always-on process to babysit |
| Static Assets | The built React SPA, served from the same Worker | One deploy, one URL, no separate hosting |

D1 and Durable Objects look similar (both are "SQLite") but solve
opposite problems. That distinction is the heart of the system, and it
gets its own section below.

## 3. Follow a request through the system

**A read (cheap and fast).**
`GET /api/v1/workspaces/abc` arrives at the nearest Cloudflare location.
Hono validates the session, reads one indexed row from D1 (under 1 ms),
and returns JSON. Total: a few milliseconds.

**A write to shared data.**
`POST /api/v1/account` validates the body with Zod, writes one row to
D1 (a few milliseconds — writes are durably persisted across multiple
locations, which is why they cost more than reads), and returns.

**A write to hot partitioned data.**
`POST /api/v1/rooms/xyz/messages` does *not* touch D1. The Worker
computes the Durable Object ID for room `xyz` and forwards the write to
that room's private database. Room `xyz` and room `abc` write at the
same time on different machines without ever contending. This is the
move that removes the write ceiling: there is no single line everyone
stands in.

**A realtime connection.**
A browser opens a WebSocket to a Durable Object (one object per room).
The object holds the socket open, broadcasts each message to its own
sockets, and persists to its own SQLite. Thousands of sockets per object;
objects scale by room count. No separate realtime service exists.

**A file upload.**
The browser asks the API for permission; the API mints an R2 presigned
URL and returns it; the browser uploads *directly to R2*, bypassing the
Worker entirely. Large files never flow through request CPU limits.

**A background job.**
Sending 10,000 emails: the API pushes 10,000 messages onto the Queue
and returns instantly. Queue consumers drain them in batches. If the job
needs steps ("send, wait a day, send a reminder"), it is a Workflow
instead — same trigger, durable state machine underneath.

## 4. The two databases and the scaling rule

Think of databases as checkout lanes.

- **D1 is one super-fast lane.** Every query to it is processed one at a
  time. Throughput follows one formula (published by Cloudflare):

  > max queries/sec = 1 / average query duration

  Indexed reads take under 1 ms, so roughly ~1,000 reads/sec. Small
  writes take several ms, so roughly ~200–500 writes/sec. A single D1
  comfortably serves thousands of simultaneously active users on a
  typical read-mostly workload — and then writes get tight.

- **Durable Objects give each partition its own lane.** Every object has
  its own SQLite database (up to 10 GB) and its own throughput, computed
  by the same formula — but lanes run in parallel. 4,000 active
  workspaces means 4,000 independent databases. If the app as a whole
  needs 2,000 writes/sec, each lane handles 0.5/sec and sits 99% idle.

The scaling rule in `AGENTS.md` is just this idea as a design law:

> Global, relational, cross-entity data lives in D1. High-write
> partitioned state lives in Durable Objects, one object per partition
> key chosen at plan time. Reads scale via D1 read replication.

Partition keys are decided during planning (Phase 1), not during an
emergency rewrite. Objects are created automatically the first time an
ID is addressed — there is no provisioning step, so "scaling" is
something that happens, not something someone does.

What this cannot do: heavy analytics that join across all partitions at
write time. That workload wants a warehouse, and no partition trick
fixes it. It is also rare.

## 5. Background work: which tool for which job

```
  Is it triggered by a request and must finish before responding?
    YES → do it inline in the request (keep it to 1–2 storage calls)
    NO ↓
  Does it run on a schedule?
    YES → Cron Trigger (which may enqueue queue messages or start a workflow)
    NO ↓
  Is it one independent unit of work (send email, resize image, webhook)?
    YES → Queue
    NO → it has steps, waits, or retries → Workflow
```

The shared principle: the request path stays thin. Anything slow,
flaky, or bursty moves behind one of these three, so traffic spikes
become queue depth (which drains) instead of timeouts (which fail).

## 6. How it scales as users grow

| Stage | Load | What changes |
|---|---|---|
| First users → thousands active | Fits one D1 (~200–500 writes/s, ~1,000 reads/s) | Nothing |
| Reads getting heavy | Past ~1k reads/s on the primary | Turn on D1 read replication (a config change); add indexes |
| Writes getting heavy | Past one D1's write rate | Nothing architectural: hot entities already live in Durable Objects per the rule, so load spreads itself; Queues smooth bursts |
| Tens of thousands concurrent | Thousands of writes/s globally | Same architecture. Per-partition load stays trivial (see math in section 4). If the *shared* D1 itself ever gets hot, split it by tenant — same code, more databases |

There is no step in this table that says "migrate to a bigger
database." That absence is the point of the design.

## 7. The cost model: counts are free, activity is cheap

There is no per-workspace, per-object, or per-database charge. An idle
object costs only its stored bytes. On the Workers Paid plan ($5/month
base), the metered lines are approximately:

- Stored data: 5 GB included, then ~$0.20/GB-month (Durable Objects)
- Object requests: 1M/month included, then ~$0.15/million
- Rows written: 50M/month included, then ~$1.00/million ← the line to watch
- Rows read: 25B/month included, then ~$0.001/million (effectively free)
- Worker requests (the API itself): 10M/month included, then ~$0.30/million

Two worked examples for 4,000 workspaces (illustrative — check current
pricing pages before budgeting):

- **Quiet** (2 MB, ~1k requests, ~2k writes each/month): ~$6/month total.
- **Busy** (50 MB, ~100k requests, ~50k writes each/month, serving tens
  of thousands of concurrent users): ~$250–300/month including the API
  Worker's own requests — about 7 cents per workspace per month.

Cost grows linearly with real activity. The one cost discipline, also in
`AGENTS.md`: rows written dominates at scale, so never rewrite rows
needlessly and batch related writes.

## 8. Local dev vs production

`doppler run -- wrangler dev` reproduces the whole system on a laptop:
D1 becomes a local SQLite file, R2 is emulated, Durable Objects persist
locally. It never touches Cloudflare (unless `--remote` is passed,
which requires asking first). The agent develops and tests against the
same bindings the production code uses — the only difference is where
the bytes live.

## 9. Limits and failure modes cheat sheet

- One D1 database: 10 GB max (cannot be raised), 2 MB max row, 100 KB
  max statement, 100 bound parameters per query.
- D1 overload: excess concurrency queues, then fails with an
  `overloaded` error — the signal that hot state belongs in Durable
  Objects.
- No `PRAGMA journal_mode` on D1: storage-level settings belong to the
  managed environment, not to user code.
- Durable Object storage: 10 GB per object; objects hibernate when idle
  (no duration billing while hibernated).
- Large multi-row `UPDATE`/`DELETE` jobs must be chunked (~1,000 rows
  at a time) to stay within execution limits.

## 10. Why this shape — the design principles

1. **One vendor, one deploy.** A single `wrangler deploy` ships the API,
   the frontend, the database bindings, and the background topology. An
   agent can hold the entire system in context.
2. **Stateless compute, partitioned state.** The Worker scales forever
   because it remembers nothing; state scales because it is partitioned
   by design, not by later heroics.
3. **Thin requests, fat background.** The request path does auth plus
   one or two storage calls. Everything else is a queue message, a
   workflow, or a cron tick.
4. **Scale-to-zero billing.** Idle costs ~nothing at every layer, so
   experiments and side projects are nearly free and growth costs arrive
   only with real usage.
