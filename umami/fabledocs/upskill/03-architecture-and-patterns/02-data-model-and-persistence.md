# Data Model & Persistence

## The entities

Sixteen Prisma models ([`prisma/schema.prisma`](../../../prisma/schema.prisma)), grouped by role:

**Analytics core (the hot data):**
- `Website` (`:66`) — the tracked entity; owns everything.
- `Session` (`:34`) — a derived visitor, keyed by the hashed `session_id`; carries browser/os/device/geo.
- `WebsiteEvent` (`:99`) — one row per pageview/custom/link/pixel/performance event. Wide and denormalized.
- `EventData` (`:152`) — custom-event properties, flattened to typed rows (EAV).
- `SessionData` (`:174`) — properties attached to a session via `identify`.
- `Revenue` (`:268`) — monetary events.

**Product/tenancy:**
- `User` (`:12`), `Team` (`:197`), `TeamUser` (`:216`) — accounts and membership.
- `Report` (`:232`), `Segment` (`:253`), `Board` (`:329`) — saved analyses/dashboards.
- `Link` (`:288`), `Pixel` (`:309`) — trackable link/pixel entities.
- `Share` (`:348`) — public grants.
- `SessionReplay` (`:362`), `SessionReplaySaved` (`:386`) — replay chunks.

## The defining modeling decisions

### 1. A wide, denormalized event table

`WebsiteEvent` has ~30 columns — url, referrer, all five UTM fields, six click-id fields, event name/tag, and five performance metrics ([`schema.prisma:99-149`](../../../prisma/schema.prisma)). This is **denormalization on purpose**: analytics queries scan huge numbers of events and aggregate; joins at that scale are expensive. Keeping everything on one row trades storage and write-width for read speed.

### 2. Composite indexes that reveal the query shape

Read the indexes and you know the queries. Almost every index leads with `(website_id, created_at, X)` ([`schema.prisma:139-148`](../../../prisma/schema.prisma)):

```
@@index([websiteId, createdAt])
@@index([websiteId, createdAt, urlPath])
@@index([websiteId, createdAt, referrerDomain])
@@index([websiteId, createdAt, eventName])
...
```

Because **every** dashboard query filters by website and time first, then breaks down by one dimension. The index order matches the `WHERE website_id = ? AND created_at BETWEEN ? AND ?` + `GROUP BY X` shape you see in `getWebsiteStats.ts` and the metrics queries. Interview gold: "how would you index an events table?" → lead with tenant + time, then the breakdown dimension, and explain why.

### 3. EAV for schemaless event properties

Custom event `data` is an arbitrary JSON object, but it's stored **flattened into typed rows** in `EventData`, not as a JSON blob ([`saveEventData.ts:27-48`](../../../src/queries/sql/events/saveEventData.ts)): each leaf becomes a row with `data_key`, and the value lands in `string_value`/`number_value`/`date_value` per its `data_type`. This is the Entity-Attribute-Value pattern.

Tradeoff to articulate: EAV makes properties *queryable and filterable* (you can `WHERE data_key='plan' AND string_value='pro'`) at the cost of read-time reconstruction and more rows. A JSON/JSONB column would be simpler to write but harder to filter/aggregate efficiently. Umami chose queryability — consistent with an analytics product.

### 4. Soft deletes

`User`, `Website`, `Team`, `Link`, `Pixel` carry a nullable `deletedAt` ([`schema.prisma:21,76,204,297,317`](../../../prisma/schema.prisma)) rather than being hard-deleted. `fetchWebsite` treats a `deletedAt` website as absent ([`load.ts:15`](../../../src/lib/load.ts)). Tradeoff: recoverability and referential safety vs. every query needing to filter `deletedAt IS NULL` (a leak risk if forgotten).

## The two-store reality

The same logical data lives in **either** Postgres **or** ClickHouse, chosen by env ([`db.ts:22-36`](../../../src/lib/db.ts)). This is the biggest persistence fact in the repo.

| | Postgres (Prisma) | ClickHouse |
| --- | --- | --- |
| Role | Small/medium self-host default | Large-scale analytics |
| Shape | Normalized-ish, session table separate | Denormalized — session fields live *on the event* |
| Writes | `INSERT` via Prisma / raw SQL | `INSERT` or via Kafka buffer |
| Reads | Raw parameterized SQL | ClickHouse SQL + materialized rollups |
| Schema | `prisma/schema.prisma` | `db/clickhouse/schema.sql` |

A key structural difference: on ClickHouse there is **no separate session insert** — session attributes are denormalized onto each event, so `createSession` is skipped ([`send/route.ts:150`](../../../src/app/api/send/route.ts)). This is why `saveEvent` has two genuinely different implementations, not just two dialects.

### Materialized views for read scaling

ClickHouse pre-aggregates events into `website_event_stats_hourly` via a materialized view ([`db/clickhouse/schema.sql:94,145`](../../../db/clickhouse/schema.sql)). Dashboard queries read the hourly rollup instead of raw events when no event-column filter is present ([`getWebsiteStats.ts:111-135`](../../../src/queries/sql/getWebsiteStats.ts)). This turns "scan millions of rows" into "scan hundreds," at the cost of up-to-an-hour freshness. This is *the* read-scaling lever for analytics.

## Transactions & consistency

- **Idempotent session insert:** `INSERT ... ON CONFLICT (session_id) DO NOTHING` ([`createSession.ts:39`](../../../src/queries/sql/sessions/createSession.ts)) — safe under at-least-once writes because the id is deterministic.
- **No cross-write transaction on collection:** event + event_data + revenue are separate awaits with no wrapping transaction ([`saveEvent.ts:143-167`](../../../src/queries/sql/events/saveEvent.ts)). A crash between them leaves partial data. Acceptable for analytics (approximate by nature); it would be wrong for financial correctness.
- Prisma exposes `transaction()` ([`prisma.ts:360-362`](../../../src/lib/prisma.ts)) — used for multi-step *product* operations (e.g. team/website changes), not the high-throughput event path where the overhead isn't worth it.
- **Read replicas:** reads route to `DATABASE_REPLICA_URL` when set ([`prisma.ts:278-281`](../../../src/lib/prisma.ts)) — eventual consistency between primary and replica is the tradeoff.

## How to change the schema safely

1. **Edit `prisma/schema.prisma`**, then create a migration (`prisma migrate dev` locally; `migrate deploy` in prod via `update-db`). Migrations live in `prisma/migrations/` and are forward-only.
2. **Mirror it in ClickHouse** if the field is on the analytics path — the two schemas are independent (`db/clickhouse/`). Forgetting this is a real trap: a column added only to Prisma won't exist for ClickHouse deployments.
3. **Update `saveEvent`/read queries** — both branches. The dual implementation means a schema change is two code changes.
4. **Respect the public contract:** the `/api/send` payload shape and column widths (`URL_LENGTH`, etc., `constants.ts:255-257`) are depended on by deployed trackers. Widen, don't break.
5. **Backfill/blast radius:** adding a nullable column is low-risk; changing a type or narrowing a width can truncate existing data. Prefer additive changes.

## Drills

1. **Index reasoning:** Given the query "top 10 referrer domains for website X last 7 days," which index serves it ([`schema.prisma:142`](../../../prisma/schema.prisma))? Why does leading with `website_id, created_at` matter?
2. **EAV round-trip:** Track a custom event `purchase` with `{ plan: 'pro', amount: 40 }`. List the `EventData` rows created ([`saveEventData.ts:33-43`](../../../src/queries/sql/events/saveEventData.ts)) and their `data_type`s.
3. **Two-store change:** You add a `country_subdivision` field to events. Enumerate every file you must touch for both Postgres and ClickHouse deployments.

## Interview angle

- "How would you model and index an events table?" → wide table, tenant+time-leading composite indexes.
- "JSON blob vs. EAV vs. columns for flexible properties?" → the `EventData` tradeoff.
- "How do you scale analytical reads?" → materialized hourly rollups.
- "How do you change a schema without downtime?" → additive migrations, dual-store mirroring, public contract.
See [`../08-interview-prep/03-api-and-data-modeling-questions.md`](../08-interview-prep/03-api-and-data-modeling-questions.md).
