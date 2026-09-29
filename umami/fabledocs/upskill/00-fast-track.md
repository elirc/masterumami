# Fast Track — A Weekend With Umami

Goal: in one weekend, get oriented, trace the two flows that define this product, make one safe change, and be able to explain the collection pipeline aloud. This is the 20% of Umami that unlocks 80% of the codebase.

> Commands are marked `__verified__` (run in this environment) or `__inferred__` (read from `package.json`/config but not executed — this clone has no `node_modules`, `.env`, or database). Treat inferred commands as "should work," not "confirmed."

## 0. Install, run, test (~30 min)

Umami uses **pnpm**, **Next.js 16**, **Postgres** (or ClickHouse), and optionally **Redis/Kafka**.

```bash
pnpm install                 # __inferred__
cp .env.example .env         # __inferred__ — you must set DATABASE_URL and APP_SECRET
pnpm build-db                # __inferred__ — prisma generate + client build (package.json:29)
pnpm dev                     # __inferred__ — dotenv next dev --turbo (package.json:14)
pnpm test                    # __inferred__ — vitest run (package.json:42)
```

Minimum env (see `docker-compose.yml` for a working pair):

```
DATABASE_URL=postgresql://umami:umami@localhost:5432/umami
APP_SECRET=replace-me-with-a-random-string
```

Fastest path to a running instance: `docker compose up` — it starts Postgres 15 and the app, with a healthcheck on `/api/heartbeat` (`docker-compose.yml`). __inferred__

If you only want to *read and reason* (this weekend's real goal), you don't need any of the above running. The code is the lab.

## 1. The two flows that define Umami

Everything else is scaffolding around these.

### Flow A — Collection (write path)

A visitor loads a page → the tracker script fires → `POST /api/send` → an event lands in Postgres or ClickHouse.

Open, in order:

1. [`src/tracker/index.js`](../../src/tracker/index.js) — the ~400-line vanilla-JS script that runs in the visitor's browser. Find `getPayload` (`src/tracker/index.js:64`) and `send` (`:163`).
2. [`src/app/api/send/route.ts`](../../src/app/api/send/route.ts) — the entire server-side collection handler. Read `POST` top to bottom (`:66`).
3. [`src/lib/crypto.ts`](../../src/lib/crypto.ts) — how a **cookieless** visitor id is derived: `uuid(sourceId, ip, userAgent, sessionSalt)` (`src/lib/crypto.ts:60`) with a monthly-rotating salt (`getSalt`, `:72`).
4. [`src/lib/db.ts`](../../src/lib/db.ts) — `runQuery` (`:22`), the switch that sends the write to Prisma **or** ClickHouse **or** Kafka.
5. [`src/queries/sql/events/saveEvent.ts`](../../src/queries/sql/events/saveEvent.ts) — the actual write, with two parallel implementations.

Pause and predict before reading `send/route.ts`: *the tracker never sends a visitor id — so how does the server know two page views came from the same person?* (Answer in Flow A of [`01-codebase-cartography/05-key-flows.md`](01-codebase-cartography/05-key-flows.md).)

### Flow B — Reading metrics (read path)

Dashboard component → React Query hook → `GET /api/websites/[id]/metrics` → permission check → parameterized SQL → JSON.

Open, in order:

6. [`src/components/hooks/queries/useWebsiteMetricsQuery.ts`](../../src/components/hooks/queries/useWebsiteMetricsQuery.ts) — a representative data-fetching hook (`:12`).
7. [`src/components/hooks/useApi.ts`](../../src/components/hooks/useApi.ts) — the fetch wrapper that injects the bearer token and share headers (`:20`).
8. [`src/lib/request.ts`](../../src/lib/request.ts) — `parseRequest` (`:12`): validate with Zod → authenticate → build filters.
9. [`src/permissions/website.ts`](../../src/permissions/website.ts) — `canViewWebsite` (`:7`): the authorization boundary that stops one user reading another's data.
10. [`src/lib/prisma.ts`](../../src/lib/prisma.ts) — `rawQuery` (`:255`), where `{{param}}` placeholders become real `$1` bind parameters.

## 2. The first 10 files to open (consolidated)

If you only open ten files this weekend, open these — they are the load-bearing walls:

| # | File | Why |
| --- | --- | --- |
| 1 | `src/app/api/send/route.ts` | The product's reason to exist; the write hot path. |
| 2 | `src/lib/crypto.ts` | Cookieless identity — the privacy thesis in code. |
| 3 | `src/lib/db.ts` | The polymorphic-persistence switch. |
| 4 | `src/queries/sql/events/saveEvent.ts` | Dual Postgres/ClickHouse write. |
| 5 | `src/lib/request.ts` | Every API request's front door. |
| 6 | `src/lib/auth.ts` | Authentication (who are you?). |
| 7 | `src/permissions/website.ts` | Authorization (may you?). |
| 8 | `src/lib/prisma.ts` | Hand-written SQL engine + param binding. |
| 9 | `prisma/schema.prisma` | The 16-model data model. |
| 10 | `src/components/hooks/useApi.ts` | The client's single door to the API. |

## 3. One small, safe change

Pick **one**. Each has a tiny blast radius and touches a real code path.

- **Add a unit test** for `hasBlockedIp` covering a CIDR range match. Read [`src/lib/detect.ts:140`](../../src/lib/detect.ts) and mirror the style of [`src/lib/boards.test.ts`](../../src/lib/boards.test.ts). Run `pnpm test`. __inferred__
- **Tighten a Zod bound**: `payload.language` is capped at 35 chars (`src/app/api/send/route.ts:34`). Add a matching cap to `name` (currently `.max(50)`, `:38`) usage elsewhere, or add a test asserting the 50-char event-name truncation in `saveEvent` (`src/queries/sql/events/saveEvent.ts:131`).
- **Documentation-as-code**: add a one-line comment above `runQuery` (`src/lib/db.ts:22`) naming the three dispatch targets. Trivial, but forces you to confirm you understand it.

Whichever you pick: it must follow an existing pattern, be covered by a test, and not touch an unrelated file. That is the contribution standard the whole `06-contribution-practice/` module builds on.

## 4. One test to run

```bash
pnpm test src/lib/boards.test.ts     # __inferred__ — small, fast, no DB needed
```

`boards.test.ts` is pure logic (a compatibility matrix), so it runs without a database. Read it first (`src/lib/boards.test.ts:1`) to see how this repo writes a unit test: `import { expect, test } from 'vitest'`, one behavior per `test`, plain assertions.

## 5. Teach-back (do this out loud)

Set a 2-minute timer and explain Flow A as if to an interviewer:

> "A tiny script POSTs an event with no cookie. The server validates it with Zod, skips auth on purpose, checks a signed cache token to avoid re-looking-up the website, derives a visitor id by hashing IP + UA + a rotating salt so it's anonymous, then dispatches the write to Postgres or ClickHouse depending on config…"

If you can say that without notes and name two failure modes (Kafka silently dropping events; a forged-but-signed cache token being replay-only), you have the weekend's core. The interview version of this exact narration is in [`08-interview-prep/04-system-design-from-this-repo.md`](08-interview-prep/04-system-design-from-this-repo.md).

## What the fast path skips (come back for these)

- The **ClickHouse columnar model** and materialized views (`db/clickhouse/schema.sql`) — covered in `03-architecture-and-patterns/02-data-model-and-persistence.md`.
- **Reports**: funnels, retention, attribution — the dynamic-SQL machinery in `src/queries/sql/reports/` (`03-.../05-pattern-catalog.md`).
- **Teams, shares, and multi-tenancy** authorization depth (`03-.../03-validation-auth-and-permissions.md`).
- **Session replay** (rrweb) ingestion and playback.
- The **React 19 / React Compiler** rendering details (`02-.../02-framework-mental-models.md`).

Next step: if you have an interview coming, jump to [`08-interview-prep/07-two-week-cram-plan.md`](08-interview-prep/07-two-week-cram-plan.md). Otherwise, continue to [`01-codebase-cartography/`](01-codebase-cartography/README.md).
