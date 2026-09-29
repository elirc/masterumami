# File Reading Order

Thirty files in a deliberate sequence. Reading a codebase randomly wastes time; this path builds understanding layer by layer. Each entry says why it matters, what to look for, and what to ignore. Junior/mid/senior paths at the end tell you where to stop.

## The core spine (read these first, in order)

| # | File | Why it matters | Look for | Ignore for now |
| --- | --- | --- | --- | --- |
| 1 | `package.json` | The project's table of contents: scripts, deps, what it builds. | The `build` script's sub-steps; ClickHouse/Kafka/Redis deps signal optional backends. | Exact version numbers. |
| 2 | `prisma/schema.prisma` | The data model — 16 entities and their relationships. | `WebsiteEvent`, `Session`, `Website`, the composite indexes. | ClickHouse-only columns. |
| 3 | `src/lib/constants.ts` | The shared vocabulary: roles, permissions, event/entity types. | `ROLES`, `PERMISSIONS`, `ROLE_PERMISSIONS` (`:164-217`), `EVENT_TYPE`. | Chart colors, browser tables. |
| 4 | `src/app/api/send/route.ts` | The write hot path — the product itself. | Zod schema, `skipAuth`, identity derivation, `saveEvent`. | Performance-metric branch first pass. |
| 5 | `src/lib/crypto.ts` | Cookieless identity in ~30 lines. | `uuid` v5 hashing (`:60`), `getSalt` (`:72`). | AES encrypt/decrypt internals. |
| 6 | `src/lib/db.ts` | The persistence dispatcher. | `runQuery` (`:22`); the BigInt patch (`:8`). | Nothing — it's tiny. |
| 7 | `src/queries/sql/events/saveEvent.ts` | The dual-backend write. | Relational vs. ClickHouse branches; child writes. | Every column name. |
| 8 | `src/lib/request.ts` | Every API request's front door. | `parseRequest` order (`:12`); the suffixed-filter re-add (`:33`). | `getQueryFilters` cohort details first pass. |
| 9 | `src/lib/auth.ts` | Authentication. | `checkAuth` two token paths (`:17`); `hasPermission` (`:76`). | Redis reconnection nuance. |
| 10 | `src/permissions/website.ts` | Authorization — the IDOR guard. | `canViewWebsite` precedence (`:7`). | Transfer functions first pass. |

## The read path & query engine

| # | File | Why it matters | Look for | Ignore |
| --- | --- | --- | --- | --- |
| 11 | `src/lib/prisma.ts` | The hand-written SQL engine + client. | `rawQuery` param binding (`:255-282`); `pagedRawQuery` (`:307`); `globalThis` singleton (`:418`). | `parseFilters` cohort branch first pass. |
| 12 | `src/queries/sql/getWebsiteStats.ts` | A representative aggregation. | The grouped subquery; index-aligned `where`. | ClickHouse materialized-view branch. |
| 13 | `src/queries/sql/sessions/getWebsiteSessions.ts` | A paginated raw query. | `pagedRawQuery` usage; hardcoded `order by`. | ClickHouse branch. |
| 14 | `src/components/hooks/useApi.ts` | The client's single door to the API. | Bearer + share headers (`:25-33`). | — |
| 15 | `src/components/hooks/queries/useWebsiteMetricsQuery.ts` | The React Query pattern. | `queryKey` composition (`:22`); `enabled` guard. | — |
| 16 | `src/store/app.ts` | Global client state. | Zustand store; `setUser`/`setConfig`. | — |

## The infrastructure & optional backends

| # | File | Why it matters | Look for | Ignore |
| --- | --- | --- | --- | --- |
| 17 | `src/lib/redis.ts` | Cache-aside + tombstone + rate-limit helper. | `fetch` (`:84`), `__DELETED__`, `rateLimit` (`:72`, unused). | Reconnection promise plumbing. |
| 18 | `src/lib/load.ts` | Cache-aside wrappers for hot entities. | `fetchWebsite` TTL (`:6`). | — |
| 19 | `src/lib/clickhouse.ts` | The columnar query builder. | `{param:Type}` binding; date formats. | Every SQL helper. |
| 20 | `src/lib/kafka.ts` | The fire-and-forget producer. | `sendMessage` swallowing errors (`:87`). | SASL config. |
| 21 | `src/lib/detect.ts` | Geo/device/bot/IP detection. | `getClientInfo` (`:126`); `hasBlockedIp` CIDR (`:140`). | Provider header list details. |
| 22 | `src/lib/jwt.ts` | Token creation/parsing. | `createSecureToken` = encrypt+sign (`:16`). | — |

## The edges (collection scripts & composition)

| # | File | Why it matters | Look for | Ignore |
| --- | --- | --- | --- | --- |
| 23 | `src/tracker/index.js` | The visitor-side script. | `send` keepalive (`:175`); `handlePathChanges` (`:98`); performance observers. | CLS/INP math first pass. |
| 24 | `src/app/(collect)/p/[slug]/route.ts` | Handler composition. | Calls `send`'s `POST` (`:60`). | — |
| 25 | `src/app/api/batch/route.ts` | Sequential per-item reprocessing. | The Request-rebuild note (`:22-33`). | — |
| 26 | `src/app/api/record/route.ts` | Replay ingest + deliberate cache bypass. | `getWebsite` direct (`:56`). | Timestamp math. |

## Auth, sharing & reports

| # | File | Why it matters | Look for | Ignore |
| --- | --- | --- | --- | --- |
| 27 | `src/app/api/auth/login/route.ts` | Two token strategies. | Redis vs. stateless branch (`:36-40`). | — |
| 28 | `src/app/api/share/[slug]/route.ts` | The second authz path. | Signed scoped token (`:94`); context guard usage. | White-label. |
| 29 | `src/lib/schema.ts` | The Zod validation surface. | `withDateRange` refine; `reportTypeSchema` discriminated union (`:279`). | Every report sub-schema. |
| 30 | `src/queries/sql/reports/getFunnel.ts` | Dynamic SQL generation. | Per-step CTE reduce (`:106-176`). | ClickHouse branch first pass. |

## Paths by level

- **Brand-new junior:** files 1–10, slowly, doing the annotation drills as you go. Stop there for week one.
- **Junior with stack familiarity:** 1–16, then the pattern catalog. Skim 17–22.
- **Mid-level onboarding:** 1–13 fast, then 23–30 for the edges and reports. Read the architecture critique after.
- **Senior review:** 4, 6, 7, 10, 11, 20, 26 first (the decisions with the biggest blast radius), then the risk register.

**Drill:** After reading files 1–10, close everything and draw the collection flow from memory. Compare with [`05-key-flows.md`](05-key-flows.md) Flow 1. The gaps are your reading-comprehension misses — reread those files, not all ten.
