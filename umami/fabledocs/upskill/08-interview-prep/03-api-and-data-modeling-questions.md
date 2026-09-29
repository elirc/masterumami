# Interview Prep: API & Data Modeling Questions

Twelve cards on REST/RPC design, validation, pagination, auth, schema design, transactions, and caching — anchored to Umami's real API and schema. Golden rule: concrete example + tradeoff + failure mode.

---

## Q1: Design the ingestion endpoint for an analytics service. Auth?

**Round:** API design. **Tests:** public-endpoint reasoning.

**Anchor:** [`send/route.ts:24-68`](../../src/app/api/send/route.ts) (`skipAuth: true`).

**Junior:** "Require an API key." **Mid adds:** ingestion is from a public script embedded on any site — there's no secret to protect, and requiring auth breaks the model. So it's unauthenticated, gated by schema validation + bot/IP filtering. **Senior adds:** the cost of open ingest is abuse exposure; you defend with rate limiting (a gap here — `redis.rateLimit` is uncalled) and bot filtering, and you accept that client-supplied fields (revenue, geo) aren't trustworthy. Being able to defend "unauthenticated on purpose" is the signal.

**Follow-up:** "How do you stop abuse then?" → bot/IP/rate limiting, edge WAF.

---

## Q2: How do you validate request input, and why if you have TypeScript?

**Round:** API/validation. **Tests:** the runtime boundary.

**Anchor:** [`request.ts:23-42`](../../src/lib/request.ts), [`schema.ts`](../../src/lib/schema.ts).

**Junior:** "TypeScript checks it." **Mid adds:** TS erases at runtime — it can't validate an HTTP body. Umami parses every request with Zod (`parseRequest`), producing a typed, coerced object; internal code then trusts the type. **Senior adds:** validation is layered (Zod at the edge, cross-field `.refine`s, write-time truncation, DB constraints), and it must happen where data *enters*. The failure mode is a bypass — e.g. the suffixed-filter re-add ([`request.ts:33-38`](../../src/lib/request.ts)) sneaks values past Zod. Validate at the boundary, and audit for holes in it.

**Follow-up:** "Where would you validate a cross-field invariant?" → `.refine` (the exactly-one-source rule, [`send/route.ts:53`](../../src/app/api/send/route.ts)).

---

## Q3: Design pagination for a large list endpoint.

**Round:** API/data. **Tests:** pagination tradeoffs.

**Anchor:** [`pagedRawQuery` prisma.ts:307-332](../../src/lib/prisma.ts); usage in [`getWebsiteSessions.ts`](../../src/queries/sql/sessions/getWebsiteSessions.ts).

**Junior:** "limit/offset." **Mid adds:** Umami uses offset pagination with a total count. Simple, supports "page N," but the count is a second scan and deep offsets are slow. **Senior adds:** for large tenants, keyset/cursor pagination on `(created_at, id)` avoids the count and stays fast at depth; the tradeoff is no random page access. Also: `pageSize` here has no upper bound ([`schema.ts:79`](../../src/lib/schema.ts)) — an unbounded page is a DoS/latency risk. The `count(*)` per request ([`prisma.ts:325`](../../src/lib/prisma.ts)) is the hidden cost.

**Follow-up:** "When is offset fine?" → small/bounded lists where users want page numbers.

---

## Q4: Prevent one user from reading another's data (IDOR).

**Round:** API/security. **Tests:** authz.

**Anchor:** [`permissions/website.ts:7`](../../src/permissions/website.ts); called in each route.

**Junior:** "Check they're logged in." **Mid adds:** that's authentication, not authorization. Umami authenticates in `parseRequest`, then each handler calls `canViewWebsite(auth, id)` against the URL id before querying. Omitting it is an IDOR. **Senior adds:** because the check is per-handler, the systemic risk is a *forgotten* one on a new route; the fix is a guardrail (wrapper/lint) so it can't be forgotten. You test it with a cross-tenant request asserting 401, not just the happy path.

**Follow-up:** "How do you make sure every route does it?" → the guardrail idea (Project S2).

---

## Q5: Design the schema for storing events and custom properties.

**Round:** data modeling. **Tests:** schema tradeoffs.

**Anchor:** `WebsiteEvent` ([`schema.prisma:99`](../../prisma/schema.prisma)), `EventData` ([`:152`](../../prisma/schema.prisma)).

**Junior:** "One events table with a JSON column." **Mid adds:** Umami uses a wide, denormalized event table (fast to aggregate, no joins) plus a separate `EventData` table storing custom properties as typed EAV rows — so properties are *filterable/aggregatable*. **Senior adds:** the EAV vs JSONB choice is the crux: EAV makes `WHERE data_key='plan'` efficient at the cost of read reconstruction and row count; JSONB is simpler to write but harder to index/aggregate at scale. Umami chose queryability, consistent with an analytics product.

**Follow-up:** "Why not JSONB?" → queryability/aggregation of properties.

---

## Q6: How would you index that events table?

**Round:** data/performance. **Tests:** index design.

**Anchor:** composite indexes at [`schema.prisma:135-148`](../../prisma/schema.prisma).

**Junior:** "Index the columns you filter on." **Mid adds:** lead every index with `(website_id, created_at, X)` because every query filters by tenant and time first, then breaks down by one dimension. **Senior adds:** column *order* in a composite index matters — the leftmost prefix must match the query's equality/range filters. Umami's indexes literally document the access pattern. The write cost: this is a write-heavy table, so each index slows inserts — you index the dimensions you actually break down by, not every column.

**Follow-up:** "Why not a single-column index on `created_at`?" → queries filter `website_id` first; a lone `created_at` index won't serve them (Review Kata 10).

---

## Q7: When do you use a transaction, and where does this app *not*?

**Round:** data/consistency. **Tests:** atomicity judgment.

**Anchor:** no transaction across event/data/revenue ([`saveEvent.ts:143-167`](../../src/queries/sql/events/saveEvent.ts)); `prisma.transaction()` ([`prisma.ts:360`](../../src/lib/prisma.ts)).

**Junior:** "Always wrap writes in a transaction." **Mid adds:** transactions give atomicity — all-or-nothing. Umami deliberately *doesn't* wrap the event + event_data + revenue writes, because analytics tolerates partial data and the hot path can't afford the overhead. It uses transactions for multi-step *product* operations. **Senior adds:** the judgment is domain-driven: partial event data is fine; a partial money transfer isn't. Recognizing where the missing transaction is acceptable (and where it wouldn't be) is the point.

**Follow-up:** "For which metric would the missing transaction matter?" → revenue, if treated as authoritative.

---

## Q8: How does this app achieve idempotent writes?

**Round:** API/reliability. **Tests:** idempotency.

**Anchor:** deterministic ids ([`crypto.ts:60`](../../src/lib/crypto.ts)); `on conflict do nothing` ([`createSession.ts:39`](../../src/queries/sql/sessions/createSession.ts)).

**Junior:** "Check if it exists first." **Mid adds:** Umami derives `sessionId` deterministically (a hash), so a retried write targets the same row, and the session insert uses `ON CONFLICT DO NOTHING` — a no-op on repeat. **Senior adds:** deterministic key + at-least-once delivery = effectively-once, the standard reliable-messaging pattern. Note the nuance: the *session* is idempotent, but the *event* row uses a fresh UUID, so a retried event can duplicate — precise knowledge of what's idempotent and what isn't.

**Follow-up:** "How does this make retries safe?" → collisions on the deterministic key.

---

## Q9: Design public share links for dashboards, safely.

**Round:** API/security. **Tests:** capability tokens.

**Anchor:** [`share/[slug]/route.ts:94`](../../src/app/api/share/[slug]/route.ts); scope check [`permissions/website.ts:12-21`](../../src/permissions/website.ts); context guard [`auth.ts:43-47`](../../src/lib/auth.ts).

**Junior:** "A public URL." **Mid adds:** Umami issues a *signed, scoped* token embedding exactly the entity ids the share grants, and `canViewWebsite` checks the requested id is in that scope — authorization without a user account. **Senior adds:** two safeguards: signing (client can't widen scope) and a context-header requirement so a leaked share token can't be used as a general API credential. The stale-scope risk: ids are embedded at creation, so later membership changes may over/under-grant until reissued.

**Follow-up:** "What stops a leaked share token from hitting other APIs?" → the context guard.

---

## Q10: How would you cache reads, and how do you invalidate?

**Round:** API/caching. **Tests:** cache coherence.

**Anchor:** cache-aside [`redis.ts:84`](../../src/lib/redis.ts); tombstone `__DELETED__`; bypass [`record/route.ts:56`](../../src/app/api/record/route.ts).

**Junior:** "Cache the result." **Mid adds:** cache-aside — check cache, miss → query → populate. Umami caches websites/sessions/pixels for 24h. Invalidation uses a `__DELETED__` tombstone so a delete/repopulate race doesn't resurrect a value. **Senior adds:** the hard part is invalidation — a value that changes without a `remove` goes stale for the TTL, which is exactly why `/api/record` *bypasses* the cache to read a fresh `replayEnabled`. The lesson: per-value, decide "cache + invalidate on write" or "read fresh." Umami is inconsistent here (one path bypasses, others don't), which is the actual defect behind a real staleness bug.

**Follow-up:** "When would you skip your own cache?" → correctness-critical fresh reads.

---

## Q11: How do you version or evolve a public API contract?

**Round:** API design. **Tests:** backward compatibility.

**Anchor:** the `/api/send` payload schema ([`send/route.ts:24-64`](../../src/app/api/send/route.ts)), depended on by deployed trackers.

**Junior:** "Just change it." **Mid adds:** the collection payload is consumed by thousands of embedded scripts you don't control — a breaking change silently breaks them. You evolve additively: add optional fields, accept both old and new during a deprecation window, never rename/remove in place. **Senior adds:** for the tracker specifically, the script is build-time templated, so old versions linger in the wild indefinitely — backward compatibility isn't optional, it's permanent. Column widths (`URL_LENGTH`, etc.) are part of the contract too.

**Follow-up:** "How would you add a required field safely?" → make it optional, backfill a default, deprecate the absence.

---

## Q12: REST vs. RPC/GraphQL for this API — why REST?

**Round:** API design. **Tests:** protocol tradeoffs.

**Anchor:** REST routes under `src/app/api/**`; client hooks in `components/hooks/queries/`.

**Junior:** "REST is standard." **Mid adds:** REST gives resource-oriented URLs, cacheable GETs, and a plain HTTP surface third parties can call — Umami exposes a public API, which REST suits. **Senior adds:** GraphQL would fix over/under-fetching but adds schema/runtime complexity and complicates the public API; tRPC gives type safety but couples client to server and excludes non-JS consumers. For a product whose API is a feature, REST's ubiquity wins; the cost is manual client typing and occasional over-fetching, both accepted.

**Follow-up:** "When would GraphQL be worth it here?" → highly variable client data needs, many clients.

---

## Drilling this module

Three cards a day, aloud, ending on a failure mode. Q4, Q5, Q6, Q10 are the deepest and most likely to escalate — over-prepare those. This module and [`04-system-design-from-this-repo.md`](04-system-design-from-this-repo.md) reinforce each other; do them in the same week. Anchors here come straight from [`../03-architecture-and-patterns/`](../03-architecture-and-patterns/README.md).
