# Architecture Critique

An honest senior review of Umami: strongest choices, real risks, and what I'd change owning this for three months. Confirmed facts are separated from hypotheses. This file doubles as system-design interview material — the "critique the design" round is exactly this exercise. Cross-referenced by [`../08-interview-prep/04-system-design-from-this-repo.md`](../08-interview-prep/04-system-design-from-this-repo.md).

## Strongest design choices (learn from these)

1. **Cookieless, hash-derived identity.** Deriving `sessionId` from IP+UA+rotating salt ([`send/route.ts:147`](../../src/app/api/send/route.ts), [`crypto.ts:60-78`](../../src/lib/crypto.ts)) is the product's whole thesis in ~30 lines: no consent banner, no PII at rest, and — because the id is deterministic — idempotent writes for free. Elegant and load-bearing.

2. **Pluggable persistence by config.** One dispatch table ([`db.ts:22`](../../src/lib/db.ts)) lets the same product run on Postgres for a hobbyist or ClickHouse+Kafka for a firehose. That the two implementations are *colocated* in each query file keeps the abstraction honest.

3. **Read scaling via materialized rollups.** Pre-aggregating events into `website_event_stats_hourly` ([`db/clickhouse/schema.sql:94,145`](../../db/clickhouse/schema.sql)) is the right lever for analytics reads and shows real operational maturity.

4. **Separated authorization module.** Keeping authz rules in `permissions/` rather than sprinkled through handlers makes them testable and auditable.

5. **Parameterized raw SQL done correctly.** The `{{param}}`→`$n` engine ([`prisma.ts:268-276`](../../src/lib/prisma.ts)) gets the expressiveness of hand-written SQL without (in the covered paths) the injection risk. Most hand-rolled SQL layers get this wrong; Umami doesn't.

6. **Index strategy that matches the query shape.** Every read is tenant+time-first, and every index leads with `(website_id, created_at, …)`. The schema *documents* the access pattern.

## Risks and tradeoffs

Grouped by severity. **[Confirmed]** = verified in code. **[Hypothesis]** = plausible, needs runtime confirmation.

### High

- **[Confirmed] Fire-and-forget Kafka = silent data loss.** A broker outage drops events while returning 200 ([`kafka.ts:87-90`](../../src/lib/kafka.ts)). For a product whose entire value is not losing events, at-most-once delivery on the primary write path is the most serious architectural weakness. → *Migration path* below.

- **[Confirmed] Per-handler authorization invites IDOR.** Authz is a manual `can*` call in each route; nothing enforces that every id-bearing route makes it. One forgotten check is a cross-tenant data leak. The risk is not any specific route (the ones reviewed are guarded) but the *absence of a systemic guarantee*.

- **[Hypothesis] No app-level rate limiting on open ingest.** `/api/send`, `/batch`, `/record` are unauthenticated and unthrottled in-app (`redis.rateLimit` exists but uncalled, [`redis.ts:72`](../../src/lib/redis.ts)). Absent an upstream WAF/proxy, they're open to volume abuse and cost amplification (especially `/batch`, which re-enters `send` per item). Needs deployment-context confirmation.

### Medium

- **[Confirmed] Latent SQL-injection surface in `pagedRawQuery`.** `order by ${orderBy}` interpolates directly ([`prisma.ts:318-323`](../../src/lib/prisma.ts)). Not currently reachable (no route passes user `orderBy`), but it's a loaded gun: one future route wiring user sort into a raw path is an injection. → allow-list the sort column.

- **[Confirmed] No atomicity across event/event_data/revenue.** Partial writes possible on crash ([`saveEvent.ts:143-167`](../../src/queries/sql/events/saveEvent.ts)). Tolerable for analytics; a landmine if revenue is ever treated as authoritative.

- **[Confirmed] Validation bypass via suffixed-filter re-add.** Numeric-suffixed filters skip Zod ([`request.ts:33-38`](../../src/lib/request.ts)). Those values flow into query building; low current risk given downstream parameterization, but it undermines "the schema is the contract."

- **[Confirmed] Client-supplied `revenue` is unauthenticated.** `eventData.revenue` comes from the public payload ([`saveEvent.ts:154`](../../src/queries/sql/events/saveEvent.ts)); anyone can POST fake revenue for a website id they know. Inherent to open ingest, but worth stating as a data-trust limit.

### Low

- **[Confirmed] Cross-cutting global patch** (`BigInt.prototype.toJSON`, [`db.ts:8`](../../src/lib/db.ts)) — precision-lossy and process-wide; a scoped serializer is cleaner.
- **[Confirmed] Divergent entity reads in permissions** (`getEntity` vs `getWebsite`, [`website.ts:23,67`](../../src/permissions/website.ts)) — maintenance drift risk.
- **[Confirmed] Thin observability** — no metrics/tracing; failures are `console.log` only.
- **[Confirmed] Type-safety gap on raw SQL** — `rawQuery` returns `any`; result interfaces are hand-maintained, unchecked against columns.

## "What I'd change owning this for 3 months"

Prioritized by (impact × reach) ÷ effort. Each with a migration path and test strategy — because a critique without a safe rollout is just complaining.

### 1. Make event delivery at-least-once (the outbox)

- **Why:** eliminates the silent-loss risk, the highest-severity issue.
- **Change:** on accept, write the event to a durable `event_outbox` table in the same transaction; a relay worker ships to Kafka with retries and deletes on ack.
- **Blast radius:** collection path only; reads untouched.
- **Migration:** additive table; run outbox *alongside* direct send behind a flag; compare counts; cut over.
- **Tests:** unit test that a stubbed Kafka failure leaves an outbox row; integration test that the relay drains it; a reconciliation check (events accepted == events in ClickHouse).
- **Why duplicates are safe:** deterministic session id + `on conflict` ([Pattern 9](05-pattern-catalog.md#pattern-9-idempotent-insert-on-conflict-do-nothing)); give events a deterministic id too if needed.

### 2. Systematize authorization

- **Why:** turn "remember to call `can*`" into "can't forget."
- **Change:** a thin `withAuthorization(capability)` wrapper for route handlers, or a lint rule that flags `[id]` routes lacking a `can*` call.
- **Blast radius:** every route, but mechanically.
- **Migration:** introduce the wrapper, adopt route-by-route, keep the manual checks until covered.
- **Tests:** a meta-test enumerating `[id]` routes and asserting each authorizes (extend the e2e suite with cross-tenant 401 assertions).

### 3. Add ingest rate limiting

- **Change:** call the existing `redis.rateLimit` ([`redis.ts:72`](../../src/lib/redis.ts)) keyed by IP+website on `/send`/`/batch`/`/record`; bound `/batch` array size (Ticket 10).
- **Blast radius:** collection; needs care not to throttle legitimate bursts.
- **Tests:** unit test the limiter branch; load-test the endpoint.

### 4. Close the `orderBy` injection surface

- **Change:** an allow-list mapping of sortable columns in `pagedRawQuery`; reject anything else.
- **Blast radius:** tiny (one function); no behavior change for current callers.
- **Tests:** unit test that a non-allow-listed `orderBy` is rejected/ignored.

### 5. Minimal observability

- **Change:** counters for events-accepted, events-persisted, kafka-errors; a reconciliation gap metric; structured error logs with an id.
- **Why:** you currently can't *see* the #1 risk happening.

## Drills

1. **Prioritize:** Re-rank the five changes for a deployment that uses **Postgres only** (no Kafka). Which drop in priority and why?
2. **Migration safety:** For change #1, describe how you'd verify no events are lost *during* the cutover, not just after.
3. **Defend a choice:** An interviewer says "cookieless identity is a hack — just use cookies." Argue Umami's side with two concrete benefits and one honest cost.

## Interview angle

This whole page *is* the "critique this architecture / what would you improve?" round. The move that reads as senior: for every criticism, give the **migration path and how you'd verify it**, and separate confirmed issues from hypotheses. Pair with [`../09-reference/risk-register.md`](../09-reference/risk-register.md) for the tabular version.
