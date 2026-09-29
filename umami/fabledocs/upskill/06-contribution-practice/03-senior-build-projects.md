# Senior Build Projects

Six projects sized from two days to four weeks. Each is something a maintainer might actually accept, with a full plan: problem, product value, design, architecture, likely files, migration, tests, security, performance, rollout/rollback, open questions, stretch. These are the "walk me through a big thing you built" interview stories.

Structure discipline: a senior doesn't start coding until problem, design, migration, test, and rollback plans exist. Practice writing those plans even if you never write the code.

---

## Project S1: Durable event delivery (outbox + relay) — ~1–2 weeks

**Problem:** On ClickHouse+Kafka deployments, a broker outage silently drops events with a 200 response ([`kafka.ts:87-90`](../../src/lib/kafka.ts)). The product's core promise — don't lose data — is violated invisibly.

**Product value:** Data integrity. For a paid analytics product, silent loss is an existential bug.

**Design checklist:** Where does "accepted" become durable? What relays it? How are duplicates handled? How do you reconcile accepted vs. persisted?

**Architecture decisions:**
- On accept, write the event to a durable `event_outbox` table (same DB transaction as any local write).
- A relay worker reads unsent rows, publishes to Kafka, marks/deletes on ack, retries with backoff.
- Duplicates are absorbed by deterministic ids + `on conflict` ([Pattern 9](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-9-idempotent-insert-on-conflict-do-nothing)); give events a deterministic id if needed.

**Likely files:** `prisma/schema.prisma` (+migration), `saveEvent.ts` (ClickHouse branch), a new `scripts/outbox-relay.ts`, `lib/kafka.ts`, config flag.

**Migration plan:** additive table; run outbox in *shadow* alongside direct send behind a flag; compare persisted counts; cut over; remove direct path later.

**Test plan:** stubbed Kafka failure leaves an outbox row; relay drains it; idempotent replay produces no duplicate events; reconciliation shows zero gap under induced failure.

**Security plan:** the outbox holds event data — same access controls as events; no new external surface.

**Performance plan:** the extra write is on the hot path — measure added latency; batch relay publishes; index the outbox on `(sent, created_at)`.

**Rollout/rollback:** flag-gated; rollback = flip the flag to direct send (current behavior). Additive migration is safe to leave in place.

**Open questions:** Does the Postgres-only path need this (it writes synchronously, so no)? Outbox retention policy?

**Stretch:** dead-letter queue + alerting on relay lag.

**Interview story:** *the* reliability story — "I turned at-most-once into at-least-once and handled duplicates with idempotency."

---

## Project S2: Systematic authorization guardrail — ~3–5 days

**Problem:** Authorization is a manual per-handler `can*` call; a forgotten check is a cross-tenant IDOR, and nothing prevents it ([architecture critique](../03-architecture-and-patterns/06-architecture-critique.md)).

**Product value:** Prevents an entire class of data-leak vulnerabilities as the API grows.

**Design checklist:** Wrapper vs. lint rule vs. both? How to adopt without a big-bang rewrite?

**Architecture decisions:**
- A `withAuthorization(capabilityFn)` higher-order handler that runs `parseRequest` + the capability check + the handler, standardizing the sequence.
- And/or a lint rule (Biome custom or a test) flagging `[id]` route files lacking a `can*` call.

**Likely files:** a new `lib/withAuthorization.ts`, incremental route adoption, a meta-test enumerating routes.

**Migration plan:** introduce the wrapper; adopt route-by-route; keep manual checks until each is covered; add the meta-test to prevent regressions.

**Test plan:** the meta-test asserts every `[id]` route authorizes; cross-tenant e2e returns 401 for each resource type.

**Security plan:** this *is* the security improvement; verify no route is accidentally made public.

**Rollout/rollback:** per-route; low risk since behavior is unchanged when the check already existed.

**Open questions:** How to express resource-id extraction generically across route shapes?

**Stretch:** codegen or types that make an unauthorized route not compile.

**Interview story:** "I converted a 'remember to check' security convention into an enforced guarantee."

---

## Project S3: Observability & reconciliation layer — ~1 week

**Problem:** No metrics/tracing; failures (especially the Kafka drop) are invisible ([observability](../05-quality-engineering/06-observability-and-operations.md)).

**Product value:** You can detect data loss, latency regressions, and attacks before customers do.

**Design decisions:** counters for events-accepted / events-persisted / kafka-errors / auth-failures; a reconciliation gap metric; structured logs with correlation ids; a metrics endpoint (Prometheus-style) or push.

**Likely files:** a new `lib/metrics.ts`, instrumentation in the collection path, `lib/response` for error ids, a metrics route.

**Migration/rollout:** additive; behind a flag; low risk.

**Test plan:** counters increment correctly; induced drop widens the reconciliation gap; error ids appear in logs and responses.

**Performance plan:** metrics on the hot path must be cheap (in-memory counters, not per-event I/O).

**Interview story:** "I made a silent-failure system observable and could finally prove data wasn't being lost."

---

## Project S4: Cursor-based pagination for large lists — ~1 week

**Problem:** `pagedRawQuery` does a full `count(*)` plus the page query every request ([`prisma.ts:325-329`](../../src/lib/prisma.ts)); with no `pageSize` cap ([`schema.ts:79`](../../src/lib/schema.ts)), big sites' session/event lists are slow.

**Product value:** Fast, stable pagination on high-volume websites.

**Design decisions:** cursor (keyset) pagination on `(created_at, id)` avoiding the count; keep offset pagination where total is truly needed; cap `pageSize`.

**Likely files:** `lib/prisma` (new paged helper), `getWebsiteSessions`/`getWebsiteEvents`, route schemas, UI pagination controls.

**Migration plan:** add cursor pagination alongside offset; migrate the heavy lists; keep offset for small ones.

**Test plan:** cursor correctness (no skips/dupes across pages); performance comparison; `pageSize` cap enforced.

**Performance plan:** measure with a seeded large dataset (`seed-data.ts`); confirm the count scan is gone via `LOG_QUERY`/`EXPLAIN`.

**Rollback:** feature-flag; revert to offset pagination.

**Interview story:** "I replaced offset+count pagination with keyset cursors and cut list latency on large tenants."

---

## Project S5: Schema-per-tenant isolation option — ~2–3 weeks

**Problem:** Enterprise customers may need hard data isolation. Umami already supports a Postgres `search_path` schema ([`prisma.ts:262-266`](../../src/lib/prisma.ts)), but it's not a first-class tenant-provisioning feature.

**Product value:** Sellable enterprise isolation guarantee.

**Design decisions:** per-tenant schema provisioning, request→schema routing, migration application per schema, connection management.

**Likely files:** provisioning script, request routing in `lib/prisma`, migration tooling, admin UI.

**Migration plan:** opt-in per tenant; provision schema, migrate, route. Existing shared-schema tenants unaffected.

**Test plan:** cross-schema isolation (tenant A can't read tenant B's schema); migrations apply to all schemas; routing correctness.

**Security plan:** the isolation *is* the security property; verify no query escapes its schema.

**Performance plan:** connection-pool implications of many schemas; migration fan-out cost.

**Open questions:** migration orchestration across N schemas; cost model.

**Interview story:** "I built enterprise-grade tenant isolation on top of Postgres schemas and handled the migration-fan-out problem."

---

## Project S6: Real-time live-visitor stream — ~2 weeks

**Problem:** The realtime view polls ([`realtime/[websiteId]/route.ts`](../../src/app/api/realtime/[websiteId]/route.ts)); true live updates need a push channel.

**Product value:** A compelling live dashboard; competitive with GA real-time.

**Design decisions:** SSE or WebSockets; a hot store (Redis counters keyed by website+minute) updated on ingest; fan-out to connected dashboards; authz on the stream.

**Likely files:** ingest increments Redis counters; a new SSE/WS route; a client hook; authz reuse.

**Migration/rollout:** additive; flag-gated; falls back to polling.

**Test plan:** counter accuracy; stream authz (only viewers of a website receive its stream); reconnection.

**Performance plan:** connection scaling (thousands of dashboards); Redis as the hot path — measure.

**Security plan:** the stream must enforce `canViewWebsite` per subscription; no cross-tenant leakage over the socket.

**Interview story:** "I designed a real-time visitor stream with Redis counters and a push channel, and handled per-subscription authorization." (Maps directly to system-design variation prompt 1.)

---

## How to use these

Pick one and write the **full plan document** (all the sections above) even if you don't implement it — that document *is* the senior interview artifact. If you do implement, S2 and S3 are the most self-contained; S1 is the highest-impact and the best story. Each maps to a system-design variation in [`../08-interview-prep/04-system-design-from-this-repo.md`](../08-interview-prep/04-system-design-from-this-repo.md).
