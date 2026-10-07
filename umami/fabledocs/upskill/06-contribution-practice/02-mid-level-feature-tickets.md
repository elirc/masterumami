# Mid-Level Feature Tickets

Twelve cross-layer tickets. Each touches schema/API/UI/tests together and **requires a design note before code**. Each includes risk and rollback sections — because a mid-level engineer plans the change, not just the diff. Estimated effort: half a day to three days.

**Required before coding, for all:** a short design note answering — what layers change, what the migration is, what could break, how to roll back, how it's tested. If you can't write that note, you don't understand the ticket yet.

---

## Ticket M1: Application-level rate limiting on collection

**Skills:** Redis, abuse prevention, cross-cutting middleware.

**Story:** As an operator, I want `/api/send`, `/batch`, `/record` rate-limited per IP+website so a single source can't flood ingest.

**Design note must cover:** key strategy (IP+website vs IP), limit/window values (justify from expected legit volume), Redis-off fallback, where the check goes (top of handler vs shared helper).

**Layers:** `lib/redis` (has `rateLimit` already, [`redis.ts:72`](../../../src/lib/redis.ts)), the three collection routes, config for limits.

**Risk:** throttling legitimate bursts (analytics is bursty); Redis dependency. **Rollback:** feature-flag the limiter; disable via env.

**Tests:** unit the limiter branch; simulate over-limit → 429.

**Acceptance:** over-limit requests get 429; under-limit unaffected; no-op when Redis disabled.

**Interview story:** "I added abuse protection to an open ingestion endpoint and had to reason carefully about not throttling legitimate traffic bursts."

---

## Ticket M2: Outbox for at-least-once Kafka delivery

**Skills:** reliability patterns, transactions, background workers. (Borders senior — see also `03-senior-build-projects.md`.)

**Story:** As an operator, I want events durably persisted before Kafka publish so a broker outage doesn't lose data.

**Design note:** the `event_outbox` table shape; write-in-same-transaction-as-accept; the relay worker; how duplicates are made safe (deterministic ids); how you'd reconcile.

**Layers:** schema (new table + migration), `saveEvent` ClickHouse branch, a relay script, `lib/kafka`.

**Risk:** outbox growth if the relay lags; duplicate delivery. **Rollback:** flag to bypass outbox and send directly (current behavior).

**Tests:** stubbed Kafka failure leaves an outbox row; relay drains it; idempotent replay.

**Interview story:** "I converted a fire-and-forget path with silent data loss into at-least-once delivery, and handled the duplicate problem with idempotent keys." (This is a top-tier STAR story.)

---

## Ticket M3: Allow-listed sortable columns for list endpoints

**Skills:** secure dynamic SQL, API design.

**Story:** As a user, I want to sort the sessions/events lists by column, safely.

**Design note:** the allow-list map (requested name → real column); how `pagedRawQuery` consumes it; rejecting unknown values.

**Layers:** `lib/prisma` (`pagedRawQuery`, [`prisma.ts:318`](../../../src/lib/prisma.ts)), the list query files, the route schemas (`sortingParams`), the UI table headers.

**Risk:** the whole point is closing the injection surface — get the allow-list airtight. **Rollback:** revert schema param; sorting reverts to hardcoded default.

**Tests:** non-allow-listed `orderBy` rejected; each allowed column sorts correctly.

**Interview story:** "I safely added user-controlled sorting to a raw-SQL endpoint by allow-listing columns, closing a latent injection surface in the process."

---

## Ticket M4: Cache invalidation on website update

**Skills:** cache coherence, understanding TTL staleness.

**Story:** As a user, when I change a website's settings (e.g. `replayEnabled`), the change takes effect immediately everywhere, not after the 24h TTL.

**Design note:** which cache keys to invalidate on update (`website:<id>`, maybe `session:*`); soft vs hard remove; why the record endpoint currently bypasses the cache and whether this lets it stop.

**Layers:** `websites/[websiteId]/route.ts` (POST/update, [`:71`](../../../src/app/api/websites/[websiteId]/route.ts)), `lib/redis`/`lib/load`.

**Risk:** missing a key → still stale; invalidating too broadly → cache churn. **Rollback:** remove the invalidation call; behavior reverts to TTL.

**Tests:** updating a website removes/refreshes its cache entry.

**Interview story:** "I fixed a 24-hour staleness bug by invalidating the cache on write instead of bypassing it on read." (Ties to [Debugging Scenario 5](../05-quality-engineering/03-systematic-debugging.md).)

---

## Ticket M5: Structured error responses with correlation ids

**Skills:** error handling, info-disclosure, observability.

**Story:** As an operator, I want 500s to return a generic message + an error id, with full detail logged server-side.

**Design note:** the error-id scheme; what's logged vs returned; updating the collection catch blocks that currently return `serializeError` to the client.

**Layers:** `lib/response`, the collection/record catch blocks ([`send/route.ts:314-320`](../../../src/app/api/send/route.ts)).

**Risk:** changing response shape may affect clients that parse errors. **Rollback:** revert response builder.

**Tests:** a forced 500 returns `{ error, id }` and logs full detail.

**Interview story:** "I stopped an unauthenticated endpoint from leaking stack traces while keeping errors debuggable via correlation ids."

---

## Ticket M6: Per-website data-export endpoint with pagination

**Skills:** API design, streaming/pagination, authz.

**Story:** As a website owner, I want to export my events as CSV.

**Design note:** pagination/streaming approach (memory bounds), authz (`canViewWebsite`), CSV shaping, rate/size limits.

**Layers:** new route under `websites/[websiteId]/export`, a query, the UI trigger.

**Risk:** unbounded export = memory/DB pressure. **Rollback:** feature-flag the route.

**Tests:** authz denies non-owner; export shape correct; large export paginates.

**Interview story:** "I built a bounded, authorized data-export path and had to defend against unbounded queries."

---

## Ticket M7: Add a new filter dimension end to end

**Skills:** touching the full stack for one feature.

**Story:** As a user, I want to filter analytics by a new dimension (pick one not yet filterable but present on events).

**Design note:** the dimension's column; `filterParams` addition ([`schema.ts:45`](../../../src/lib/schema.ts)); `FILTER_COLUMNS` mapping; both SQL branches; the UI filter control.

**Layers:** `schema.ts`, `constants.ts` (FILTER_COLUMNS), query builders, UI filter component.

**Risk:** forgetting one SQL branch → works on Postgres, breaks on ClickHouse. **Rollback:** remove the filter param + UI control.

**Tests:** filter applies in both query dialects; UI round-trips it.

**Interview story:** "I threaded a new filter dimension through validation, both SQL backends, and the UI — a lesson in keeping dual implementations in sync."

---

## Ticket M8: Login rate limiting with lockout protection

**Skills:** auth security, DoS-vs-security tradeoff.

**Story:** As a user, I want brute-force protection on login that doesn't let attackers lock me out.

**Design note:** key by IP (not username-only, to avoid targeted lockout); backoff; Redis-off behavior; uniform messaging.

**Layers:** `auth/login/route.ts`, `lib/redis`.

**Risk:** locking out real users; Redis dependency. **Rollback:** flag off.

**Tests:** N failures from an IP → throttled; different users from different IPs unaffected.

**Interview story:** "I added brute-force protection to login while avoiding the account-lockout DoS that naive per-username limiting creates." (Ties to [Review Kata 5](../04-code-reading-gym/04-review-katas.md).)

---

## Ticket M9: Bounded, validated report parameters on save

**Skills:** validation depth, discriminated unions.

**Story:** As a maintainer, I want report writes validated by the strict per-type schema, not just `anyObjectParam`.

**Design note:** apply `reportTypeSchema` ([`schema.ts:279`](../../../src/lib/schema.ts)) at the save path; migration/back-compat for existing reports with loose params.

**Layers:** report save route, `schema.ts`, tests.

**Risk:** existing saved reports with invalid params could fail validation. **Rollback:** revert to `anyObjectParam`.

**Tests:** each report type validates its params; invalid params rejected.

**Interview story:** "I tightened report validation to use an existing discriminated union, and handled backward compatibility for already-saved data."

---

## Ticket M10: Health endpoint that verifies DB writability

**Skills:** observability, health-check design.

**Story:** As an operator, I want `/api/heartbeat` (or a deeper `/health`) to confirm the DB is reachable, without becoming a false-alarm source.

**Design note:** cheap read vs write check; timeout; separate liveness vs readiness; avoid making the container healthcheck flap.

**Layers:** heartbeat route, maybe a new readiness route, `docker-compose.yml`.

**Risk:** a heavy check flaps the container. **Rollback:** revert to shallow check.

**Tests:** returns unhealthy when DB unreachable (mocked), healthy otherwise.

**Interview story:** "I deepened a health check to catch DB outages while keeping it cheap enough not to cause false restarts."

---

## Ticket M11: Migrate a magic-number/env behavior to typed config

**Skills:** config design, type safety.

**Story:** Centralize scattered `process.env` reads (e.g. collection tuning flags) into one typed config module.

**Design note:** a `config.ts` parsing/validating env once (Zod), typed access; which reads to migrate; back-compat.

**Layers:** new `lib/config`, call sites in `send/route.ts`/`detect.ts`.

**Risk:** changing env parsing could alter behavior. **Rollback:** keep old reads until parity confirmed.

**Tests:** config parses valid env; rejects invalid.

**Interview story:** "I replaced scattered, untyped env reads with a single validated config surface."

---

## Ticket M12: Add reconciliation metric (accepted vs persisted)

**Skills:** observability, the silent-loss problem.

**Story:** As an operator, I want to know when events are accepted but not persisted (the Kafka-drop gap).

**Design note:** counters at accept and at persist; how to expose them (log/metric endpoint); alert threshold.

**Layers:** collection path, a metrics surface.

**Risk:** metric overhead on the hot path. **Rollback:** remove counters.

**Tests:** counters increment correctly; a simulated drop widens the gap.

**Interview story:** "I made a silent data-loss failure observable by adding an accepted-vs-persisted reconciliation metric." (Pairs with M2.)

---

## How to work a mid-level ticket

1. Write the design note **first** — layers, migration, risk, rollback, tests. Get it reviewed before coding (that's the mid-level habit).
2. Make the smallest change that satisfies the story; resist scope creep.
3. Keep the two SQL branches in sync (the #1 mid-level trap here).
4. Test the deny/failure path, not just the happy path.
5. Write the PR per [`../07-career-and-collaboration/02-writing-prs-and-rfcs.md`](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).

Start with **M4** (cache invalidation) or **M3** (safe sorting) — both are self-contained, high-value, and produce excellent STAR stories.
