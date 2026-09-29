# System Design From This Repo: "Design a Web Analytics Platform"

This is the highest-leverage interview asset in the curriculum. Umami *is* the answer to a common system-design prompt. This page reverse-engineers it into a whiteboard exercise, shows what Umami actually chose (with anchors), and names the stronger/simpler alternatives — so you can defend real decisions instead of reciting a template.

Read [`01-codebase-cartography/05-key-flows.md`](../01-codebase-cartography/05-key-flows.md) Flow 1 first; this builds on it.

---

## The prompt

> "Design a privacy-friendly web analytics service like Google Analytics. Sites embed a script; you collect pageviews and custom events; owners see dashboards with visitor counts, top pages, referrers, and funnels."

Interviewers want to see you move through: **requirements → API → data model → write path → read path → scaling → tradeoffs**, surfacing assumptions and failure modes. Below, each step pairs "what a strong candidate says" with "what Umami actually did."

---

## Step 1 — Clarify requirements (2–3 min)

Say these out loud; don't assume.

- **Read vs write ratio?** Writes dominate massively (every pageview is a write; dashboards are viewed rarely). → optimize the write path.
- **Privacy/consent?** If cookieless, we avoid consent banners but lose durable identity. Umami chose **cookieless** — identity is a hash of IP+UA+rotating salt ([`send/route.ts:147`](../../src/app/api/send/route.ts), [`crypto.ts:60-78`](../../src/lib/crypto.ts)).
- **Accuracy tolerance?** Analytics tolerates approximate counts (sampling, bots, shared IPs). This unlocks cheap choices.
- **Multi-tenant?** Yes — users, teams, per-website isolation. → authorization is first-class.
- **Real-time?** "Near" real-time dashboards, not sub-second. Umami has a realtime endpoint with a rolling window ([`realtime/[websiteId]/route.ts`](../../src/app/api/realtime/[websiteId]/route.ts)).

Junior answer: dives straight into tables. Mid: asks read/write ratio and privacy. Senior: ties each requirement to a specific downstream choice (cookieless → no consent → salt rotation → visitor-count reset tradeoff).

---

## Step 2 — Sketch the API (3–4 min)

Two surfaces: **ingest** (public, tiny, fast) and **query** (authed, rich).

Ingest — one endpoint, unauthenticated:

```
POST /api/send
{ type: 'event'|'identify'|'performance',
  payload: { website, url, referrer, title, screen, language, ... } }
```

What Umami did: exactly this ([`send/route.ts:24-66`](../../src/app/api/send/route.ts)), plus `POST /api/batch` (many events) and `POST /api/record` (replay). Pixel (`GET /p/[slug]`) and link (`GET /q/[slug]`) variants for no-JS contexts ([Flow 4](../01-codebase-cartography/05-key-flows.md#flow-4-pixel--link-tracking--handler-composition)).

Query — authed, per-website:

```
GET /api/websites/:id/stats?startAt&endAt&filters
GET /api/websites/:id/metrics?type=url|referrer|browser...
GET /api/websites/:id/pageviews
POST /api/reports/funnel
```

What Umami did: 95 route files under `src/app/api/`, grouped by resource. Reads follow the [validate→auth→authorize→SQL pipeline](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-6-validate--authenticate--authorize-pipeline).

Key decision to voice: **ingest is unauthenticated**. Why acceptable? The site owner embeds a public script; there's no secret to protect on collection, and requiring auth would break the model. You defend abuse with bot filtering, IP blocking, and (should-have) rate limiting.

Junior: makes ingest authenticated "for security." Mid: recognizes ingest must be public and defends it with bot/IP filters. Senior: notes the missing rate limit ([`redis.rateLimit` exists but uncalled](../../src/lib/redis.ts)) and proposes edge rate-limiting.

---

## Step 3 — Data model (4–5 min)

Core entities: **event**, **session**, **website**, **user/team**.

What Umami did ([`prisma/schema.prisma`](../../prisma/schema.prisma)):

- `website_event` (`:99`) — one row per pageview/custom event, wide: url, referrer, UTM, click ids, event name, performance metrics. Denormalized on purpose for query speed.
- `session` (`:34`) — browser/os/device/geo, keyed by the hashed `session_id`.
- `event_data` (`:152`) — custom event properties flattened into typed rows (`string_value`/`number_value`/`date_value` + `data_type`). This is the EAV (entity-attribute-value) pattern for schemaless properties.
- `website`, `user`, `team`, `team_user` — tenancy.

Two modeling decisions worth discussing:

1. **Wide, denormalized event table** vs. normalized. Analytics reads aggregate over huge row counts; denormalization avoids joins. The index strategy (`schema.prisma:135-149`) is composite `(website_id, created_at, X)` — because *every* query filters by website and time first. Point at those indexes; they reveal the query shape.
2. **EAV for custom event data** (`event_data`) — flexible schemaless properties at the cost of query complexity (you reconstruct objects at read time, `flattenJSON`/`getStringValue`, [`saveEventData.ts:27-48`](../../src/queries/sql/events/saveEventData.ts)). The classic tradeoff: flexibility vs. queryability.

Junior: one `events` table with a JSON blob column. Mid: separates session from event, adds time+tenant indexes. Senior: explains EAV vs. JSONB vs. columnar, and *why the composite index leads with `website_id, created_at`*.

---

## Step 4 — The write path at scale (4–5 min)

The crux of the whole design. Writes dominate; how do you keep ingest fast and durable?

What Umami did:

- **Cookieless identity computed server-side** so the client sends nothing to store ([`send/route.ts:147`](../../src/app/api/send/route.ts)).
- **A signed client cache token** to skip the website lookup and session insert on repeat requests ([Pattern 5](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-5-client-held-signed-cache-token)).
- **Idempotent session insert** (`on conflict do nothing`, [`createSession.ts:39`](../../src/queries/sql/sessions/createSession.ts)) so at-least-once writes don't duplicate.
- **Pluggable storage**: Postgres for small deployments; **ClickHouse** (columnar, built for analytics) for large ones; **Kafka** as an ingestion buffer in front of ClickHouse ([`db.ts:22-36`](../../src/lib/db.ts), [`saveEvent.ts:259-263`](../../src/queries/sql/events/saveEvent.ts)).

The scaling story to tell: *"Start with Postgres. When write volume outgrows a row store, switch to ClickHouse — a columnar DB that eats analytical aggregations. Put Kafka in front so ingest just publishes and a consumer batches inserts, decoupling spikes from DB write speed."* That is literally Umami's progression, and you can point at the dispatch table that encodes it.

Failure mode to name (senior signal): Umami's Kafka path is **fire-and-forget** — `sendMessage` swallows errors ([`kafka.ts:87-90`](../../src/lib/kafka.ts)). A broker outage silently drops events with a 200 response. The stronger design is an **outbox** or ack-based retry for at-least-once delivery, which reintroduces duplicates — handled by the idempotent key. Walking that trade both directions is a strong-answer move.

Junior: writes each event straight to the DB synchronously. Mid: adds a queue to buffer spikes. Senior: discusses at-least-once vs. at-most-once, the fire-and-forget data-loss risk here, and how the deterministic id makes retries idempotent.

---

## Step 5 — The read path & aggregation (3–4 min)

Dashboards aggregate millions of rows into a few numbers.

What Umami did:

- **Hand-written parameterized SQL** for aggregations an ORM can't tune ([Pattern 2](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-2-parameterized-raw-sql-template)); `getWebsiteStats` computes pageviews/visitors/visits/bounces/total-time in one grouped query ([`getWebsiteStats.ts:40-69`](../../src/queries/sql/getWebsiteStats.ts)).
- **ClickHouse materialized views** for pre-aggregation: `website_event_stats_hourly` rolls events up hourly so dashboards read summaries, not raw events (`db/clickhouse/schema.sql:94,145`; used in `getWebsiteStats.ts:125`).
- **Read replicas** — reads route to `DATABASE_REPLICA_URL` when set ([`prisma.ts:278-281`](../../src/lib/prisma.ts)).
- **Server-state caching** on the client via React Query keyed by params ([Pattern 11](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-11-server-state-cache-react-query-keyed-by-params)).

The lever to name: **pre-aggregation**. Raw events are expensive to scan repeatedly; a materialized hourly rollup turns a dashboard query from "scan 10M rows" into "scan 240 rollup rows." That's the single biggest read-scaling idea and Umami implements it.

Junior: `SELECT count(*) ... GROUP BY` over raw events every time. Mid: adds indexes and caches results. Senior: introduces materialized/rollup tables and explains the freshness-vs-cost tradeoff (hourly rollups lag by up to an hour).

---

## Step 6 — Multi-tenancy & authorization (2–3 min)

Every query is scoped to a website the caller may access.

What Umami did: authenticate in `parseRequest`, then an explicit `canViewWebsite(auth, id)` per handler ([`permissions/website.ts:7`](../../src/permissions/website.ts)) checking admin → ownership → team membership → share-token scope. The IDOR guard is that single call ([Flow 2, step 6](../01-codebase-cartography/05-key-flows.md#flow-2-reading-a-metric--the-authorized-read-path)).

Senior signal: note that authz is per-handler, so the systemic risk is a *forgotten* check on a new route — and propose a defense (a wrapper that requires a capability, or a lint rule).

---

## Step 7 — Tradeoffs summary (say this to close)

| Decision | Umami's choice | Cost | Anchor |
| --- | --- | --- | --- |
| Identity | Cookieless hash + rotating salt | Visitor counts reset per period; shared IPs merge | `crypto.ts:72` |
| Ingest auth | Public endpoint | Needs bot/IP/rate defenses | `send/route.ts:68` |
| Storage | Postgres → ClickHouse → +Kafka | Two SQL dialects to maintain | `db.ts:22` |
| Delivery | Fire-and-forget to Kafka | Silent data loss on outage | `kafka.ts:87` |
| Read scaling | Hourly materialized rollups | Up to 1h staleness | `clickhouse schema :94` |
| Event props | EAV (`event_data`) | Complex read reconstruction | `saveEventData.ts:27` |

---

## Variation prompts (practice these)

1. **"Now make it real-time (sub-second live visitor count)."** Discuss: a hot store (Redis counters keyed by website+minute), WebSockets/SSE to push, and how Umami's rolling-window realtime endpoint approximates this today. What breaks at 10k concurrent dashboards?
2. **"Now support 10× write traffic."** Discuss: Kafka partitioning by `website_id`, batched ClickHouse inserts, backpressure, and why the fire-and-forget path must become an acked/outboxed one. Where's the bottleneck — network, DB write, or session hashing?
3. **"Add strict per-tenant data isolation for enterprise."** Discuss: schema-per-tenant (Umami already supports a `search_path` schema, [`prisma.ts:262-266`](../../src/lib/prisma.ts)) vs. row-level `website_id` filtering vs. separate databases; the migration and query-routing cost of each.
4. **"Guarantee no event loss."** Discuss: replace fire-and-forget with an outbox table + relay, or synchronous acks; the duplicate problem that follows and how the deterministic `event_id`/`session_id` (Pattern 9) absorbs it.
5. **"Add funnels over arbitrary steps."** You can literally show them: [`getFunnel.ts`](../../src/queries/sql/reports/getFunnel.ts) generates per-step CTEs joined within a time window. Discuss the SQL-generation vs. app-side-iteration tradeoff.

---

## What junior / mid / senior sound like overall

- **Junior**: one events table, synchronous writes, `GROUP BY` on read, auth on everything. Correct-ish but doesn't scale and misses privacy.
- **Mid**: separates session/event, indexes on tenant+time, buffers writes with a queue, caches reads, scopes queries per tenant, names the read/write asymmetry.
- **Senior**: leads every choice from a requirement, quantifies the write/read asymmetry, chooses columnar + rollups for reads and a durable queue for writes, calls out the fire-and-forget data-loss risk and the idempotency that makes retries safe, and states each tradeoff's cost out loud.

**Practice drill:** Whiteboard this end-to-end in 20 minutes from the prompt alone, *then* open Flow 1 and check which of Umami's real decisions you missed. Re-run until your unaided design matches or beats Umami's on at least the write path and read aggregation.
