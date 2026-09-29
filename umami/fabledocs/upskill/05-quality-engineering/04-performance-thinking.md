# Performance Thinking

The senior rule: **measure first.** Most performance "fixes" target the wrong thing. This page maps Umami's performance domains, where the likely hotspots are (with anchors), and how to find each class of problem before touching code.

## The performance domains here

| Domain | Where cost lives | How to measure |
| --- | --- | --- |
| **Server request** | Collection hot path, report queries | Server timing logs, APM |
| **Database** | Aggregations over `website_event` | `LOG_QUERY=1` + `EXPLAIN ANALYZE` |
| **Client render** | Dashboard charts/tables | React DevTools Profiler |
| **Network** | API payloads, tracker size | DevTools Network |
| **Bundle** | Dashboard JS, tracker script | `next build` output, bundle analyzer |
| **Memory/connections** | DB/Redis clients | pool metrics, the `globalThis` singleton |
| **Cache** | Redis hit rate | Redis metrics |

## Database: the primary hotspot

Analytics is DB-bound. The likely offenders and how to spot them:

- **Unbounded scans.** `getWebsiteStats`/`getWebsiteSessions` scan `website_event` filtered by `website_id` + date. A wide date range = a big scan. The composite `(website_id, created_at, …)` indexes ([`schema.prisma:139-148`](../../prisma/schema.prisma)) serve these — confirm with `EXPLAIN` that they're used, not sequential-scanned.
- **The double count in pagination.** `pagedRawQuery` runs a `count(*)` over the subquery *and* the page query ([`prisma.ts:325-329`](../../src/lib/prisma.ts)) — two scans per list request. For large result sets, the count dominates. Cursor pagination avoids it.
- **Unbounded `pageSize`.** No max ([`schema.ts:79`](../../src/lib/schema.ts)) means a client can request an enormous page — Ticket 5.
- **Dynamic report SQL.** Funnels generate CTEs per step ([`getFunnel.ts`](../../src/queries/sql/reports/getFunnel.ts)); each step is another self-join within a time window. Cost grows with step count (bounded to 8 by Zod).

**How to find an N+1 or slow query:** set `LOG_QUERY=1` ([`prisma.ts:256`](../../src/lib/prisma.ts)) or `DEBUG=umami:prisma`, exercise the flow, count the queries and read the slow one's `EXPLAIN ANALYZE`. A loop emitting one query per iteration is an N+1; a single query with a huge sort/seq-scan is a missing/unused index or an unbounded range.

## The read-scaling lever: pre-aggregation

The biggest DB win isn't tuning a query — it's **not scanning raw events**. ClickHouse deployments read `website_event_stats_hourly`, an hourly materialized rollup ([`db/clickhouse/schema.sql:94,145`](../../db/clickhouse/schema.sql); used at [`getWebsiteStats.ts:125`](../../src/queries/sql/getWebsiteStats.ts)), turning millions of rows into hundreds. The tradeoff is up-to-1h staleness. When someone asks "how would you make dashboards faster?", pre-aggregation is the headline answer.

## Server: the collection hot path

`/api/send` runs per pageview, so its per-request cost matters at scale:

- **The cache token** skips the website lookup and session insert on repeat requests ([`send/route.ts:104-112`](../../src/app/api/send/route.ts)) — a real latency win.
- **Serial awaits** in `saveEvent` add up ([`saveEvent.ts:143-167`](../../src/queries/sql/events/saveEvent.ts)); event_data + revenue could run concurrently after the event insert (safe optimization, see Review Kata 2).
- **Synchronous crypto:** `pbkdf2Sync` in `getKey` ([`crypto.ts:16`](../../src/lib/crypto.ts)) is CPU-blocking — fine for login, but it's only on the token-encrypt path, not per-event (the per-event `uuid`/`hash` are cheap SHA/UUID ops). Worth knowing which crypto is on which path.

## Client: render & bundle

- **Charts/tables** re-render on data changes; React 19's compiler auto-memoizes ([`package.json:136`](../../package.json)), reducing manual `useMemo`. Find expensive renders with the Profiler — look for components re-rendering when their props didn't change (often a broad Zustand selector or an unstable prop).
- **`keepPreviousData`** ([`useWebsiteMetricsQuery.ts:40`](../../src/components/hooks/queries/useWebsiteMetricsQuery.ts)) avoids empty flashes on refetch — a perceived-performance win.
- **Bundle:** the dashboard ships React + charts + maps (`react-simple-maps`, `chart.js`). Check `next build` output for large chunks; heavy, rarely-used views (maps, replay player) are `import()`-splitting candidates.

## Network & the tracker

The tracker is downloaded by *every visitor* to *every tracked site*, so its size is a public-performance concern. It's vanilla JS, no framework, Rollup-minified to one file ([`build-tracker`](../../package.json)). Events are small JSON POSTs with `keepalive`. The recorder batches (100 events/10s, [`recorder/index.js:30-31`](../../src/recorder/index.js)) to avoid chatty requests. When discussing tracker perf: small size + batching + sampling are the levers.

## Memory & connections

The `globalThis` singleton for Prisma/Redis/ClickHouse/Kafka ([`prisma.ts:418`](../../src/lib/prisma.ts)) prevents connection-pool exhaustion under dev hot-reload — a memory/connection-leak defense ([Pattern 12](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-12-global-singleton-via-globalthis)). In production, one pool is created normally.

## Finding-a-hotspot checklist

- [ ] Reproduce with a realistic dataset (small data hides the problem).
- [ ] `LOG_QUERY=1` and count queries — is it an N+1?
- [ ] `EXPLAIN ANALYZE` the slow query — seq scan? unused index? huge sort?
- [ ] Is the input unbounded (date range, page size)? Bound it.
- [ ] React Profiler — which component re-renders and why?
- [ ] Network tab — payload size, request count.
- [ ] `next build` — any oversized chunk?
- [ ] Only *then* change code, and re-measure.

## Drill

Given "the sessions list times out for a big site," write the exact investigation order (not the fix). Compare with [Debugging Scenario 4](03-systematic-debugging.md#scenario-4-the-sessions-list-is-slow-and-sometimes-times-out-for-big-sites). Then name the two DB-cost contributors you'd expect to find (unbounded range + redundant count) and cite the lines.

## Interview angle

- "This endpoint is slow — how do you approach it?" → measure first: LOG_QUERY, EXPLAIN, then bound inputs; don't reflexively add indexes.
- "How do you scale analytical reads?" → pre-aggregated rollups.
- "How would you find an N+1?" → query count under LOG_QUERY / a loop with an await.
- "How do you keep a widely-embedded script fast?" → small, batched, sampled.
