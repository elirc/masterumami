# System Map

## What kind of project is this?

A **single Next.js 16 application** (App Router) — not a monorepo, though it builds several artifacts from one tree: the web app, a browser **tracker** script, a **recorder** (session-replay) script, and a shareable component bundle (`tsup`, `package.components.json`). One `package.json`, one `src/`.

Runtime surfaces:
- **Server** — Next.js API route handlers under `src/app/api/` (95 `route.ts` files) plus the collection routes under `src/app/(collect)/`.
- **Browser (dashboard)** — React 19 pages/components under `src/app/(main)/` and `src/components/`.
- **Browser (visitor)** — the standalone `src/tracker/index.js` and `src/recorder/index.js`, which run on *other people's* sites, not the dashboard.
- **Data** — Postgres (via Prisma) or ClickHouse (via `@clickhouse/client`), optional Redis cache and Kafka queue.

## Shape (ASCII)

```
                          Visitor's website
                 ┌───────────────────────────────┐
                 │  <script src=".../script.js">  │  src/tracker/index.js
                 │  (+ optional recorder)         │  src/recorder/index.js
                 └───────────────┬───────────────┘
                                 │ POST /api/send, /batch, /record
                                 │ GET  /p/[slug] (pixel), /q/[slug] (link)
                                 ▼
   ┌───────────────────────────────────────────────────────────────┐
   │                     Next.js server (src/app)                   │
   │                                                                │
   │  Collection routes ──► lib/request (validate)                  │
   │  api/* routes       ──► lib/auth (authN) ──► permissions/ (authZ)
   │                              │                                  │
   │                              ▼                                  │
   │                    queries/  (data access)                     │
   │        ┌──────────────┬───────────────┬───────────────┐        │
   │        │ prisma/      │ sql/ (raw)     │ lib/db runQuery│        │
   │        └──────┬───────┴──────┬────────┴───────┬────────┘        │
   └───────────────┼──────────────┼────────────────┼────────────────┘
                   ▼              ▼                ▼
              Postgres        ClickHouse         Kafka ──► ClickHouse
              (Prisma)       (columnar)        (buffer)
                   ▲
                   │  optional cache
                Redis  (lib/redis, lib/load)

   Dashboard (browser): src/app/(main)/** + src/components/**
        │  React Query hooks (components/hooks/queries/*)
        └─► lib/fetch ─► GET/POST /api/*  (same server above)
```

## Ownership map

| Layer | Owns | Must NOT own | Where |
| --- | --- | --- | --- |
| Tracker/recorder | Capturing events in the visitor's browser; batching; unload-safety | Any server secret; any DB knowledge | `src/tracker/`, `src/recorder/` |
| Collection routes | Accepting anonymous events; deriving identity; writing events | User authorization (they're public) | `src/app/api/send`, `/batch`, `/record`, `src/app/(collect)/` |
| API routes | HTTP shape, calling validate→auth→authorize→query | Business SQL, authorization *rules* | `src/app/api/**/route.ts` |
| `lib/request` + `lib/auth` | Input validation, authentication | Per-resource authorization | `src/lib/request.ts`, `src/lib/auth.ts` |
| `permissions/` | Authorization rules (who may do what) | HTTP, SQL | `src/permissions/` |
| `queries/` | Data access (Prisma models + raw SQL) | HTTP, auth | `src/queries/prisma/`, `src/queries/sql/` |
| `lib/db`, `lib/prisma`, `lib/clickhouse`, `lib/kafka`, `lib/redis` | Backend connections + the SQL engine | Product logic | `src/lib/` |
| Dashboard components | Rendering, local UI state, data hooks | Direct DB access; secrets | `src/app/(main)/`, `src/components/` |
| `store/` | Global client state (user, theme, share) | Server state (that's React Query) | `src/store/` |

## Public interfaces vs. private internals

**Public (contracts you must not break casually):**
- The **collection API** (`POST /api/send`) — thousands of deployed tracker scripts depend on its payload shape ([`send/route.ts:24-64`](../../src/app/api/send/route.ts)). This is the most sacred contract in the repo.
- The **tracker script's `data-*` attributes** and `window.umami.track/identify` API ([`tracker/index.js:387-392`](../../src/tracker/index.js)).
- The **REST API** under `/api/*` — used by the dashboard and by third-party API consumers.
- The **database schema** — migrations are forward-only; see `prisma/migrations/`.

**Private (refactor freely with a green build):**
- Everything in `src/lib/` internals, `src/queries/` implementations, component internals, the store.

## Directory quick-reference

| Path | What's there |
| --- | --- |
| `src/app/api/` | 95 REST route handlers, grouped by resource (websites, teams, users, reports, ...) |
| `src/app/(collect)/` | Pixel (`p`) and link (`q`) tracking routes |
| `src/app/(main)/` | The authenticated dashboard UI (websites, reports, settings, ...) |
| `src/lib/` | Cross-cutting infra: auth, crypto, db engines, request pipeline, detect, redis |
| `src/permissions/` | Authorization predicates per resource |
| `src/queries/prisma/` | ORM-based reads/writes (CRUD entities) |
| `src/queries/sql/` | Hand-written analytical SQL (metrics, reports, sessions) |
| `src/components/` | Reusable UI + `hooks/` (incl. 60+ React Query hooks) |
| `src/store/` | Zustand global stores |
| `src/tracker/`, `src/recorder/` | Visitor-side scripts |
| `prisma/` | Postgres schema + migrations |
| `db/clickhouse/`, `db/postgresql/` | ClickHouse schema/migrations, PG data-migrations |
| `cypress/` | e2e tests |
| `scripts/` | Build/ops scripts (check-db, seed, geo, tracker update) |

## Senior noticing

- The **two SQL worlds** (`queries/prisma/` for CRUD entities, `queries/sql/` for analytics, each with dual Postgres/ClickHouse implementations) are the defining structural fact. CRUD goes through Prisma models; analytics goes through hand-written parameterized SQL. Knowing which is which tells you where to look for any change.
- `permissions/` being a *separate top-level directory* (not folded into routes) signals that authorization is treated as a first-class concern — and hints that the risk is a route forgetting to call it.
- The tracker/recorder living in the same repo as the server means a payload-shape change has to be coordinated across the client script and the server schema — a real cross-boundary contract.

**Drill:** Without opening them, predict which directory holds (a) the funnel report SQL, (b) the rule that a team-member can delete a website, (c) the code that turns a bearer token into a user. Then verify: `src/queries/sql/reports/`, `src/permissions/website.ts`, `src/lib/auth.ts`.
