# Pattern Catalog

Sixteen patterns that recur in Umami, each as a card you can recognize in *any* codebase. The goal is recognition and articulation, not name-dropping. Every card has at least one real anchor.

Index:
1. [Polymorphic persistence via dispatch table](#pattern-1-polymorphic-persistence-via-dispatch-table)
2. [Parameterized raw-SQL template](#pattern-2-parameterized-raw-sql-template)
3. [Cache-aside with soft-delete tombstone](#pattern-3-cache-aside-with-soft-delete-tombstone)
4. [Deterministic hashed identity (cookieless)](#pattern-4-deterministic-hashed-identity-cookieless)
5. [Client-held signed cache token](#pattern-5-client-held-signed-cache-token)
6. [Validate → authenticate → authorize pipeline](#pattern-6-validate--authenticate--authorize-pipeline)
7. [Separated permission module (capability functions)](#pattern-7-separated-permission-module-capability-functions)
8. [Handler-as-function composition](#pattern-8-handler-as-function-composition)
9. [Idempotent insert (`on conflict do nothing`)](#pattern-9-idempotent-insert-on-conflict-do-nothing)
10. [Discriminated-union schema validation](#pattern-10-discriminated-union-schema-validation)
11. [Server-state cache (React Query) keyed by params](#pattern-11-server-state-cache-react-query-keyed-by-params)
12. [Global singleton via `globalThis`](#pattern-12-global-singleton-via-globalthis)
13. [Dynamic SQL generation (funnel CTEs)](#pattern-13-dynamic-sql-generation-funnel-ctes)
14. [Fire-and-forget side effect](#pattern-14-fire-and-forget-side-effect)
15. [Feature/billing gate at the boundary](#pattern-15-featurebilling-gate-at-the-boundary)
16. [Encrypted (not just signed) token](#pattern-16-encrypted-not-just-signed-token)

---

## Pattern 1: Polymorphic persistence via dispatch table

**Problem it solves:** The same logical operation must run against different storage engines (relational Postgres vs. columnar ClickHouse) without callers knowing which.

**General shape:** A function returns a map keyed by backend; a dispatcher picks one entry based on config and invokes it.

**Real example:** [`src/lib/db.ts:22-36`](../../src/lib/db.ts) — `runQuery({ [PRISMA]: fn, [CLICKHOUSE]: fn, [KAFKA]: fn })` picks by `CLICKHOUSE_URL`/`DATABASE_URL`.

**Second example:** Every write/read query uses it: [`saveEvent.ts:65-70`](../../src/queries/sql/events/saveEvent.ts), [`getWebsiteStats.ts:20-24`](../../src/queries/sql/getWebsiteStats.ts).

**Why this implementation works:** Each query file colocates both implementations (`relationalQuery` and `clickhouseQuery`), so the two stay visible side by side; the dispatcher is trivial and central.

**Failure modes:** The two implementations can silently drift (different bounce logic, different columns). There's no test asserting parity. Adding a third backend touches every query file.

**Use it when:** you genuinely support multiple backends. **Avoid it when:** you have one backend and are speculatively abstracting — that's premature.

**Interview angle:** "How would you support both Postgres and a columnar store?" — this is a concrete, defensible answer.

**Drill:** Find one query where the relational and ClickHouse branches compute a metric differently (hint: compare `getWebsiteStats.ts` bounce handling across branches). Is it a bug or intentional?

---

## Pattern 2: Parameterized raw-SQL template

**Problem it solves:** You need hand-tuned analytical SQL (aggregations, window logic) that an ORM can't express well, without opening a SQL-injection hole.

**General shape:** SQL written with named placeholders; a small engine rewrites placeholders to positional bind parameters and passes values separately.

**Real example:** [`src/lib/prisma.ts:255-282`](../../src/lib/prisma.ts) — `rawQuery` turns `{{name::type}}` into `$1,$2,...` (`:268-276`) then calls `$queryRawUnsafe(query, ...params)`.

**Second example:** ClickHouse equivalent uses `{name:Type}` binding — [`src/lib/clickhouse.ts:73-99`](../../src/lib/clickhouse.ts).

**Why this implementation works:** Values never touch the SQL string; only the *shape* of the query is dynamic. `$queryRawUnsafe` is "unsafe" about the text, not the values.

**Failure modes:** Anything interpolated *outside* the placeholder mechanism is unprotected — e.g. `order by ${orderBy}` in `pagedRawQuery` ([`prisma.ts:318-323`](../../src/lib/prisma.ts)). Today no route feeds user input there, but it's a latent injection surface (see `09-reference/risk-register.md`).

**Use it when:** ORM expressiveness is the bottleneck. **Avoid it when:** a query builder covers the case — you lose type safety here.

**Interview angle:** "Is raw SQL always unsafe?" — No; explain parameter binding vs. string concatenation with this exact code.

**Drill:** In `prisma.ts:268-276`, explain why the regex captures `::type` and appends it after `$n`. What would break if it didn't?

---

## Pattern 3: Cache-aside with soft-delete tombstone

**Problem it solves:** Cache expensive lookups (website, session, pixel) while still being able to *invalidate* an entry so a stale value isn't served after a delete.

**General shape:** On read, check cache; miss → query source, populate cache. On delete, write a tombstone value rather than just removing the key, so a concurrent repopulate doesn't resurrect it.

**Real example:** [`src/lib/redis.ts:84-102`](../../src/lib/redis.ts) — `fetch()` is cache-aside; `remove(key, soft=true)` writes `__DELETED__`; `fetch` returns `null` for a tombstone (`:87`).

**Second example:** [`src/lib/load.ts:6-20`](../../src/lib/load.ts) — `fetchWebsite` wraps `getWebsite` in `redis.client.fetch(..., 86400)`.

**Why this implementation works:** The tombstone closes the race where a delete and a cache-miss-repopulate interleave and the deleted value comes back.

**Failure modes:** 24h TTL means a non-tombstoned change (e.g. `replayEnabled`) can be stale for a day — which is exactly why `/api/record` bypasses this cache ([`record/route.ts:56`](../../src/app/api/record/route.ts)). Choosing between "cache it" and "read fresh" is a per-flow decision.

**Use it when:** reads dominate and the source is expensive. **Avoid it when:** the value must always be current and you can't tolerate TTL staleness.

**Interview angle:** "How do you invalidate a cache safely?" and "when would you skip your own cache?"

**Drill:** Explain the specific interleaving the `__DELETED__` tombstone prevents that a plain `del()` would not.

---

## Pattern 4: Deterministic hashed identity (cookieless)

**Problem it solves:** Attribute events to a "visitor" without cookies or stored PII, for privacy and GDPR-friendliness.

**General shape:** Identity = a stable hash of quasi-identifiers (IP, UA) plus a rotating salt, so the same visitor maps to the same id *within a window* but can't be tracked across windows or reversed to PII.

**Real example:** [`src/app/api/send/route.ts:143-147`](../../src/app/api/send/route.ts) — `sessionId = uuid(sourceId, ip, userAgent, sessionSalt)`; `getSalt` rotates monthly by default ([`crypto.ts:72-78`](../../src/lib/crypto.ts)). `uuid(...)` is a v5 hash ([`crypto.ts:60-66`](../../src/lib/crypto.ts)).

**Second example:** `visitId = uuid(sessionId, visitSalt)` with an hourly salt ([`send/route.ts:145,168`](../../src/app/api/send/route.ts)).

**Why this implementation works:** No cookie to consent to; rotating the salt breaks cross-period linkage; the hash is one-way.

**Failure modes:** Salt rotation causes visitor counts to reset at the boundary (a returning visitor looks new next month). Shared IPs (offices, NAT) collapse distinct people into one id. It's a deliberate accuracy-for-privacy trade.

**Use it when:** privacy/consent-free tracking is a product requirement. **Avoid it when:** you need durable per-user identity — use an explicit id (`distinctId`, `send/route.ts:147`).

**Interview angle:** "How would you count unique visitors without cookies?" — a standout answer.

**Drill:** What happens to a visitor's `sessionId` at midnight on the 1st of the month with `SALT_ROTATION=month`? Trace `getSalt`.

---

## Pattern 5: Client-held signed cache token

**Problem it solves:** Skip repeated server work (website lookup, session insert) on a high-frequency endpoint by letting the client carry back a small, tamper-proof token of prior state.

**General shape:** Server signs `{state}` → client stores it → client returns it on the next request → server verifies signature and trusts the state to skip work.

**Real example:** [`src/app/api/send/route.ts:104-112`](../../src/app/api/send/route.ts) reads `x-umami-cache`; [`:311-313`](../../src/app/api/send/route.ts) issues a fresh one; the tracker stores it ([`tracker/index.js:189`](../../src/tracker/index.js)).

**Why this implementation works:** The token is signed with `secret()` (`jwt.createToken`), so a client can't forge new state — only replay what the server issued.

**Failure modes:** The client now holds server state, which can go stale (a deleted website cached in the token). Replay within validity is possible. This is the reason `/api/record` doesn't trust cached website state.

**Use it when:** an endpoint is hot and the skipped work is safe to skip given a valid token. **Avoid it when:** the cached state is security- or correctness-critical.

**Interview angle:** "How do you reduce per-request work on a write-heavy endpoint?"

**Drill:** Could an attacker mint a cache token for a website they don't own? Read `jwt.parseToken` (`jwt.ts:8-14`) and explain why or why not.

---

## Pattern 6: Validate → authenticate → authorize pipeline

**Problem it solves:** Every mutating/reading endpoint needs the same three gates in the same order; doing them ad hoc invites skipped checks.

**General shape:** A shared function validates input and authenticates; the handler then performs an explicit authorization check for the specific resource.

**Real example:** [`src/lib/request.ts:12-53`](../../src/lib/request.ts) does validate + authenticate; handlers add authz, e.g. [`websites/[websiteId]/route.ts:28`](../../src/app/api/websites/[websiteId]/route.ts) `if (!(await canViewWebsite(auth, websiteId))) return unauthorized()`.

**Why this implementation works:** Validation and authentication are centralized (hard to forget); authorization stays in the handler because only the handler knows *which* resource.

**Failure modes:** Because authz is per-handler, a new route can forget the `can*` call — the classic IDOR. There's no framework-level enforcement that every id-bearing route authorizes.

**Use it when:** always, for authed APIs. **Avoid it when:** never skip a layer; if a route is intentionally public, make that explicit (`skipAuth: true`).

**Interview angle:** "Walk me through your request-handling middleware." Distinguish authN from authZ clearly.

**Drill:** Audit three `[id]` routes for a missing `can*` call. (Start: `websites/[websiteId]/daterange/route.ts`.) Is each one guarded?

---

## Pattern 7: Separated permission module (capability functions)

**Problem it solves:** Authorization logic scattered across handlers becomes inconsistent. Centralize it as named capability predicates.

**General shape:** A `permissions/` module exports `canDoX(auth, ...ids)` functions, each encoding the ownership/role rules for one capability.

**Real example:** [`src/permissions/website.ts`](../../src/permissions/website.ts) — `canViewWebsite`, `canUpdateWebsite`, `canDeleteWebsite`, `canTransferWebsiteToTeam`, etc., all following admin → ownership → team-role.

**Second example:** [`src/permissions/team.ts`](../../src/permissions/team.ts), `src/permissions/index.ts` (barrel).

**Why this implementation works:** Each predicate is testable in isolation and reused across routes; the pattern (admin bypass, then owner, then team-role via `hasPermission`) is uniform.

**Failure modes:** `canViewWebsite` and `canUpdateWebsite` re-fetch the entity independently (`getEntity` vs `getWebsite`) — subtle divergence risk. The role-permission matrix ([`constants.ts:186-217`](../../src/lib/constants.ts)) is a second source of truth that must stay in sync.

**Use it when:** more than a couple of resources have ownership rules. **Avoid it when:** a single trivial check — inline is fine.

**Interview angle:** "Where does your authorization logic live and why not in the controller?"

**Drill:** Compare the team-role branch of `canUpdateWebsite` (`website.ts:77-81`) with `canDeleteWebsite` (`:105-108`). What permission does each require, and is the matrix consistent?

---

## Pattern 8: Handler-as-function composition

**Problem it solves:** Two endpoints need the collection logic without duplicating it.

**General shape:** Import another route's exported handler and call it with a synthetic request.

**Real example:** [`p/[slug]/route.ts:60`](../../src/app/(collect)/p/[slug]/route.ts) and [`q/[slug]/route.ts:58`](../../src/app/(collect)/q/[slug]/route.ts) call `POST(req)` from `send/route.ts`; [`batch/route.ts:36`](../../src/app/api/batch/route.ts) calls `send.POST` per item.

**Why this implementation works:** Next.js route handlers are plain async functions — reusing them avoids reimplementing validation and persistence.

**Failure modes:** Tight coupling to `send`'s request contract; a schema change ripples silently. Reconstructing a `Request` has footguns (the `batch` route documents one: private-field errors, so it rebuilds headers, `batch/route.ts:22-33`).

**Use it when:** the logic is genuinely identical and lives behind a stable contract. **Avoid it when:** you'd be better served extracting a plain function both routes call (a real refactor kata).

**Interview angle:** "How do you avoid duplicating endpoint logic?" — plus the honest cost.

**Drill:** Extract a `collectEvent(payload, request)` function and describe how `send`, `p`, `q`, and `batch` would call it instead. What's the blast radius?

---

## Pattern 9: Idempotent insert (`on conflict do nothing`)

**Problem it solves:** The same session may be "created" many times (every event derives the same hashed `sessionId`); the insert must not fail or duplicate.

**General shape:** `INSERT ... ON CONFLICT (pk) DO NOTHING`.

**Real example:** [`src/queries/sql/sessions/createSession.ts:39`](../../src/queries/sql/sessions/createSession.ts).

**Why this implementation works:** Because `sessionId` is deterministic (Pattern 4), repeated inserts collide on the primary key and are no-ops — the write is idempotent.

**Failure modes:** If session *attributes* change (e.g. geo refined later), the first write wins and updates are ignored. Fine for analytics; wrong for mutable records.

**Use it when:** at-least-once delivery meets a deterministic key. **Avoid it when:** you need last-write-wins semantics — use upsert.

**Interview angle:** "What does idempotency mean and how do you get it in a database write?"

**Drill:** Why does the ClickHouse path *not* call `createSession` at all (`send/route.ts:150`)? What replaces it?

---

## Pattern 10: Discriminated-union schema validation

**Problem it solves:** One endpoint accepts several report shapes; each needs different required fields.

**General shape:** A Zod `discriminatedUnion` on a `type` field routes to the correct sub-schema.

**Real example:** [`src/lib/schema.ts:279-289`](../../src/lib/schema.ts) — `reportTypeSchema` unions goal/funnel/journey/performance/retention/utm/revenue/attribution/breakdown, each its own object with a literal `type`.

**Why this implementation works:** The discriminant makes validation precise and errors specific; adding a report type is a localized change.

**Failure modes:** The persisted `parameters` column is `Json` (`schema.prisma:239`) and the base save schema uses `anyObjectParam` (`schema.ts:276`), so the strict per-type validation only applies where `reportTypeSchema` is actually used — a place to check.

**Use it when:** a polymorphic payload has a stable tag field. **Avoid it when:** shapes barely differ — one optional-heavy schema is simpler.

**Interview angle:** "How do you validate a request that can take several shapes?"

**Drill:** Trace where `funnelReportSchema` (`schema.ts:171-196`) is enforced vs. where a report is merely saved with `anyObjectParam`. Is every write validated by the strict union?

---

## Pattern 11: Server-state cache (React Query) keyed by params

**Problem it solves:** Dashboard components need cached, deduped, refetchable server data without hand-rolling loading/error state.

**General shape:** A `useQuery` hook with a `queryKey` array that includes every input that changes the result; `queryFn` calls the API.

**Real example:** [`useWebsiteMetricsQuery.ts:21-42`](../../src/components/hooks/queries/useWebsiteMetricsQuery.ts) — key = `['websites:metrics', {websiteId, startAt, endAt, ...filters, ...params}]`, `placeholderData: keepPreviousData`.

**Second example:** 60+ sibling hooks in `src/components/hooks/queries/`.

**Why this implementation works:** The key captures all inputs, so changing a filter fetches fresh data while `keepPreviousData` avoids UI flicker. `enabled: !!websiteId` defers until ready.

**Failure modes:** Omit an input from the key → stale data shown for new inputs. Over-broad keys → cache thrash. Objects in keys must be stable/serializable.

**Use it when:** caching server state in a React app. **Avoid it when:** the value is truly local UI state — use `useState`/Zustand (`store/app.ts`).

**Interview angle:** "How do you cache API data in React and invalidate it?"

**Drill:** If you added a `timezone` param to the metrics request but forgot to add it to `queryKey`, what bug appears? Reproduce it mentally against `:22-31`.

---

## Pattern 12: Global singleton via `globalThis`

**Problem it solves:** In dev, Next.js hot-reload re-imports modules, which would create many DB/Redis clients and exhaust connections.

**General shape:** Store the client on `globalThis` in non-production and reuse it.

**Real example:** [`prisma.ts:390,413,418`](../../src/lib/prisma.ts) (`globalThis[PRISMA] ??= ...`); [`redis.ts:144-151`](../../src/lib/redis.ts); [`clickhouse.ts:41-43`](../../src/lib/clickhouse.ts); [`kafka.ts:44-46`](../../src/lib/kafka.ts).

**Why this implementation works:** One client survives hot reloads; production creates it once normally.

**Failure modes:** Hides state across reloads; a stale global can mask config changes until restart. Typing `globalThis[X]` loses type safety.

**Use it when:** long-lived clients in a hot-reloading dev server. **Avoid it when:** the object is cheap or request-scoped.

**Interview angle:** "Why do Next.js apps put the Prisma client on `globalThis`?"

**Drill:** What error would you expect in dev *without* this pattern after 50 hot reloads? (Hint: connection pool.)

---

## Pattern 13: Dynamic SQL generation (funnel CTEs)

**Problem it solves:** A funnel report has a variable number of steps; the SQL must be generated per request.

**General shape:** Reduce over the steps, emitting one CTE per step that joins the previous step within a time window, then union the counts.

**Real example:** [`src/queries/sql/reports/getFunnel.ts:95-176`](../../src/queries/sql/reports/getFunnel.ts) — `getFunnelQuery` builds `level1..levelN` CTEs; step values bound as `{{i}}` params (`:163`), filters bound too (`:69-90`).

**Why this implementation works:** Steps and filter values are still parameterized (Pattern 2); only the *structure* (number of CTEs) is dynamic. Zod caps steps at 2–8 (`schema.ts:193-194`), bounding the generated SQL.

**Failure modes:** Generated SQL is hard to read and to test; a mistake in the reduce affects every funnel. The wildcard handling (`*` → `like %`, `:116-119`) changes the operator dynamically — subtle.

**Use it when:** query structure legitimately depends on input. **Avoid it when:** a fixed query with an `IN`/array would do.

**Interview angle:** "How would you implement a funnel report?" — a strong, concrete system-design sub-question.

**Drill:** Explain how the 30-minute-window join at `:152-155` restricts step *N* to events after step *N-1*. What bounds the window's length?

---

## Pattern 14: Fire-and-forget side effect

**Problem it solves:** Push an event to a downstream system (Kafka) without blocking or failing the request on downstream errors.

**General shape:** Send asynchronously; catch and log errors instead of propagating them.

**Real example:** [`src/lib/kafka.ts:66-91`](../../src/lib/kafka.ts) — `sendMessage` wraps `producer.send` in try/catch and only `console.log`s on failure (`:87-90`).

**Why this implementation works (and doesn't):** It keeps the collection path fast and resilient to broker hiccups. But it means **a Kafka outage silently drops events** with a success response — no outbox, no retry, no visibility.

**Failure modes:** Data loss without an error. No dead-letter queue. This is the canonical "at-most-once" pitfall.

**Use it when:** the side effect is truly optional and loss is acceptable. **Avoid it when:** the event is the product (it is, here) — an outbox or ack-based retry would be safer.

**Interview angle:** "What are the reliability tradeoffs of fire-and-forget vs. an outbox?" — cite this exact code as the "before."

**Drill:** Sketch the minimal change to turn this into at-least-once delivery. What new failure mode does that introduce (hint: duplicates → Pattern 9)?

---

## Pattern 15: Feature/billing gate at the boundary

**Problem it solves:** Some capabilities (session replay, extended history) are gated by plan or config; the check must happen before doing the work.

**General shape:** Early-return guard reading a plan/config flag at the top of the handler.

**Real example:** [`record/route.ts:67-77`](../../src/app/api/record/route.ts) — `if (process.env.CLOUD_MODE) { ...require business subscription... }`. Also `setWebsiteDate` clamps history for non-subscribers ([`request.ts:92-109`](../../src/lib/request.ts)).

**Why this implementation works:** Gates live at the boundary, close to the request, and fail closed (`403`/clamped range).

**Failure modes:** The same gate must be repeated per feature; a new gated endpoint can forget it. Config-driven behavior (`CLOUD_MODE`) makes local behavior differ from cloud — a testing gotcha.

**Use it when:** plan/config changes what a request may do. **Avoid it when:** the gate belongs in the data layer or a shared guard — don't scatter it.

**Interview angle:** "How do you implement plan-based feature gating?"

**Drill:** List every behavior that changes when `CLOUD_MODE` is set. (Grep `CLOUD_MODE`.) Which are security-relevant?

---

## Pattern 16: Encrypted (not just signed) token

**Problem it solves:** Sometimes you want the token body to be *unreadable* by the client, not merely tamper-evident.

**General shape:** Sign a JWT, then encrypt the whole thing; reverse on read.

**Real example:** [`src/lib/jwt.ts:16-26`](../../src/lib/jwt.ts) — `createSecureToken` = `encrypt(createToken(...))`; `parseSecureToken` decrypts then verifies. Encryption is AES-256-GCM ([`crypto.ts:18-46`](../../src/lib/crypto.ts)).

**Contrast:** The share/cache tokens use plain `createToken`/`parseToken` (signed only) because their contents (granted ids) aren't secret — [`share/[slug]/route.ts:94`](../../src/app/api/share/[slug]/route.ts).

**Why this implementation works:** Auth tokens (carrying `userId`/`role`) are encrypted so a client can't read or diff them; public-scope tokens are only signed.

**Failure modes:** Encryption adds CPU and hides useful debugging info; using it where signing suffices is over-engineering. Losing `APP_SECRET` invalidates all tokens.

**Use it when:** the token body is sensitive. **Avoid it when:** you only need integrity — sign, don't encrypt.

**Interview angle:** "Signing vs. encryption for tokens — when each?" Point at the two token types in this repo.

**Drill:** Why is the collection `cache` token only *signed* while the login token is *encrypted*? Justify from what each carries.

---

## Using this catalog

- In review katas ([`04-code-reading-gym/04-review-katas.md`](../04-code-reading-gym/04-review-katas.md)), name the pattern a diff violates.
- In interviews, "I've seen X implemented as…" + one of these cards is a mid-level signal. The interview module maps several of these directly to question cards.
