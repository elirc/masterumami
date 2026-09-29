# Verification Log

This log records what was actually inspected while building this curriculum, so every teaching claim is traceable. Anchors elsewhere in the docs were confirmed by re-opening files at the moment of writing.

Convention: `__verified__` = command run in this environment and output observed. `__inferred__` = derived from reading files/config, not executed (this repo was cloned without `node_modules`, `.env`, or a database, so runtime commands were not executed).

## Environment facts

- Repo: `umami` v3.1.0 (`package.json:3`), MIT license, Next.js 16 App Router, React 19, TypeScript 5.9. __verified__ (read `package.json`)
- No `node_modules`, no `.env`, no running DB in the clone. Build/test/dev commands are therefore documented as __inferred__. __verified__ (`ls`, no `node_modules`)
- Git: repository has **no commits** (`git log` → "does not have any commits yet"). No history to mine for interview stories; stories are sourced from the code as it stands. __verified__
- Package manager: pnpm (`pnpm-lock.yaml`, `pnpm-workspace.yaml` present). __verified__
- Linter/formatter: Biome (`biome.json`, scripts `lint`/`format`/`check`). __verified__ (read `package.json:47-49`)
- Test runner: Vitest for unit (`vitest.config.ts`, `test` script), Cypress for e2e (`cypress/`). __verified__

## Files read in full or in relevant part

| Area | File | What was confirmed |
| --- | --- | --- |
| Collection hot path | `src/app/api/send/route.ts` | Zod schema, skipAuth, cache token, bot/IP checks, session+visit derivation, event/identify/performance branches |
| Collection composition | `src/app/(collect)/p/[slug]/route.ts`, `q/[slug]/route.ts` | Pixel (gif) and link (redirect) routes internally call `send` POST |
| Batch | `src/app/api/batch/route.ts` | Sequential per-item Request reconstruction, error collection |
| Replay ingest | `src/app/api/record/route.ts` | Cache-token session, direct `getWebsite` (bypasses Redis), CLOUD_MODE billing gate |
| Auth | `src/lib/auth.ts` | Bearer + share token, Redis auth-key vs stateless token, admin flag |
| Request pipeline | `src/lib/request.ts` | `parseRequest`: validate → auth → filters; suffixed-filter re-add hack |
| Authorization | `src/permissions/website.ts` | Entity-ownership + team-role checks, admin bypass, share-token match |
| Roles | `src/lib/constants.ts:164-217` | ROLES, PERMISSIONS, ROLE_PERMISSIONS matrix |
| Persistence dispatch | `src/lib/db.ts` | `runQuery` routes to Prisma / ClickHouse / Kafka by env |
| Event write | `src/queries/sql/events/saveEvent.ts` | Relational vs ClickHouse branches, substring truncation, revenue side effect |
| Event data | `src/queries/sql/events/saveEventData.ts` | JSON flatten → typed columns |
| Raw SQL engine | `src/lib/prisma.ts` | `{{param}}::type` → `$n`, `parseFilters`, cohort/bounce/date SQL, `pagedRawQuery` |
| ClickHouse engine | `src/lib/clickhouse.ts` | Parallel query builder, `{param:Type}` binding, date formats |
| Crypto/identity | `src/lib/crypto.ts` | `uuid()` v5 hashing, `getSalt` rotation, AES-GCM encrypt/decrypt |
| Redis client | `src/lib/redis.ts` | cache-aside `fetch`, `__DELETED__` tombstone, `rateLimit` (unused by routes) |
| Geo/device | `src/lib/detect.ts` | provider headers → MaxMind fallback, `hasBlockedIp` CIDR |
| Kafka | `src/lib/kafka.ts` | fire-and-forget producer, error swallowed |
| Tracker | `src/tracker/index.js` | IIFE, pushState hook, click tracking, Web Vitals observers, `beforeSend` |
| Recorder | `src/recorder/index.js` | rrweb sampling, flush by count/interval |
| Client data layer | `src/components/hooks/useApi.ts`, `queries/useWebsiteMetricsQuery.ts`, `useUpdateQuery.ts` | React Query wrapper, share headers, query keys |
| Store | `src/store/app.ts`, `src/lib/client.ts` | Zustand global store, localStorage auth token |
| Schema (DB) | `prisma/schema.prisma` | 16 models, index strategy, soft-delete columns, relationMode="prisma" |
| Schema (API) | `src/lib/schema.ts` | Zod param groups, discriminated report union, funnel/segment schemas |
| Funnel report | `src/queries/sql/reports/getFunnel.ts` | Dynamic CTE generation per step, window join |
| Website CRUD | `src/app/api/websites/[websiteId]/route.ts` | GET/POST/DELETE, share upsert, unique-constraint catch |
| Login | `src/app/api/auth/login/route.ts` | password check, token issuance |
| Share resolve | `src/app/api/share/[slug]/route.ts` | entity→token, white-label |
| Realtime | `src/app/api/realtime/[websiteId]/route.ts` | window-bounded filter build |
| Tests | `src/lib/boards.test.ts`, `cypress/e2e/login.cy.ts` | unit style, e2e `data-test` selectors |
| Infra | `docker-compose.yml`, `next.config.ts` | Postgres 15 service, CSP header, tracker env injection |

## Confirmed observations (facts, not bugs)

1. `/api/send`, `/api/batch`, `/api/record` all call `parseRequest(..., { skipAuth: true })` — collection endpoints are intentionally unauthenticated. `src/app/api/send/route.ts:68`.
2. Trust of the client-supplied `x-umami-cache` JWT lets `/api/send` skip the website lookup and session insert. It is signed with `secret()`, so it cannot be forged, only replayed within its content. `src/app/api/send/route.ts:104-112`, `src/lib/jwt.ts`.
3. Session/visitor identity is a **hash**, not a cookie: `uuid(sourceId, ip, userAgent, sessionSalt)` with a monthly-rotating salt. `src/app/api/send/route.ts:147`, `src/lib/crypto.ts:60-78`.
4. Persistence is polymorphic: one logical write dispatches to Postgres (Prisma) **or** ClickHouse, and optionally Kafka, by environment variable — never both stores at once. `src/lib/db.ts:22-36`.
5. `saveEvent` → `saveEventData` → `saveRevenue` run as sequential awaits inside the request; there is no outbox/transaction spanning them on the ClickHouse path. `src/queries/sql/events/saveEvent.ts:143-167`.
6. Kafka `sendMessage` swallows errors (`console.log` then returns undefined) — a broker failure silently drops the event. `src/lib/kafka.ts:87-90`.
7. `/api/record` deliberately calls `getWebsite` directly instead of the Redis-cached `fetchWebsite`, to avoid a stale `replayEnabled` flag. `src/app/api/record/route.ts:56-57`.

## Possible risks (labelled — investigate, not confirmed bugs)

1. **Latent SQL-injection surface in `pagedRawQuery`.** `order by ${orderBy} ${direction}` interpolates `orderBy` directly into SQL. `src/lib/prisma.ts:318-323`. Currently **not exploitable**: the analytics list routes (`/websites/[id]/sessions`, `/events`) hardcode `order by ... desc` in the SQL and their Zod schemas omit `sortingParams`, so `filters.orderBy` is always `undefined`. Risk is that a future route wiring user `orderBy` into a raw path would introduce injection. Verified by reading `getWebsiteSessions.ts`, `getWebsiteEvents.ts`, and the route schemas.
2. **No application-level rate limiting on collection.** `redis.rateLimit` exists (`src/lib/redis.ts:72-82`) but `grep` finds no caller in `src/app/api`. `/api/send` is open to volume abuse absent an upstream proxy. Investigate whether infra handles this.
3. **`/api/batch` is sequential and unbounded in item count** beyond Zod's array validation; each item re-enters the full `send` handler. Potential latency/DoS amplification. `src/app/api/batch/route.ts:21-46`.

## Final anchor-verification pass

After drafting, key anchors were re-opened and confirmed against the live source (line-for-line): `send/route.ts:147` (sessionId derivation), `db.ts:8` (BigInt patch), `prisma.ts:318-323` (orderBy interpolation), `kafka.ts:87-90` (swallowed error), `createSession.ts:39` (`on conflict do nothing`), `record/route.ts:56` (cache-bypass comment), `permissions/website.ts:7` (canViewWebsite), `schema.ts:279` (discriminatedUnion), `constants.ts:186` (ROLE_PERMISSIONS), `schema.prisma:99` (WebsiteEvent), `request.ts:33-38` (suffixed-filter re-add), `jwt.ts:16` (createSecureToken). All matched. Line numbers elsewhere were written against the same clone and should be re-checked if the repo advances (line numbers drift).

## Not covered / uncertainties

- Did not execute any build, migration, test, or dev command (no DB/deps in clone). All command docs are `__inferred__` from scripts in `package.json`.
- ClickHouse and Kafka code paths read but not run; correctness of aggregate SQL against a live ClickHouse not verified.
- `@umami/react-zen` is an external design-system dependency; its internals were not inspected, only its usage surface.
- The `docs/` folder already contained three unrelated learning trees (`architectural-cartographer`, `mission-learning-path`, `user-story-build-path`); these were left untouched, and this curriculum lives separately under `fabledocs/upskill/`.
