# Trace Tables

Following a value across files is the core skill of reading unfamiliar code. Fill each table's blank rows *before* reading the completed version. Columns: **step, file/line, value shape, owner, transformation, risk.**

Six traces, covering UI→API, persistence, auth, error, a report, and a cache path.

---

## Trace 1 — A metric request, browser to SQL (UI→API)

**Scenario:** the dashboard renders "top pages"; follow the request.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `useWebsiteMetricsQuery.ts:33` | `{ startAt, endAt, ...filters, type:'url' }` | UI hook | Assembled into GET params | Missing key input → stale cache |
| 2 | `useApi.ts:46-49` | Request + `Authorization` header | client | Adds bearer + share headers | Token from localStorage (XSS-readable) |
| 3 | `request.ts:26` | `query` (validated) | server | Zod parse of query string | Suffixed filters re-added unvalidated (`:33`) |
| 4 | `request.ts:44-50` | `Auth` | server | `checkAuth` resolves user | 401 if unauth |
| 5 | `websites/[websiteId]/metrics/route.ts` | boolean | server | `canViewWebsite(auth, id)` | **IDOR if omitted** |
| 6 | `request.ts:111` | `QueryFilters` | server | date range + filters + segment | Segment loaded from DB |
| 7 | `prisma.ts:232` | `{ filterQuery, queryParams }` | query layer | filters → SQL fragments + params | — |
| 8 | `prisma.ts:268-276` | SQL + `$n` params | engine | `{{param}}` → bind params | Values bound, not concatenated |
| 9 | route | JSON rows | server | serialize (BigInt→Number) | precision loss >2^53 |

**Self-check:** Could you name, unprompted, the two lines that are the security-critical hops? (Steps 5 and 8.) If not, reread Flow 2.

---

## Trace 2 — An event write, payload to row (persistence)

**Scenario:** a pageview arrives; follow it to a stored row (Postgres path). Fill the blanks first.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `send/route.ts:74` | validated `payload` | server | destructure | client-controlled |
| 2 | `send/route.ts:98` | `sourceId` | server | `websiteId \|\| pixelId \|\| linkId` | assumes exactly-one (refine) |
| 3 | `send/route.ts:147` | `sessionId` | server | hash(source, ip, ua, salt) | shared IP collisions |
| 4 | `send/route.ts:151-165` | session row | query | `createSession` on-conflict | idempotent |
| 5 | `send/route.ts:225` | `SaveEventArgs` | server | build event object | — |
| 6 | `db.ts:22` | dispatch | engine | pick Prisma branch | silent undefined if misconfigured |
| 7 | `saveEvent.ts:107-141` | `website_event` row | query | truncate + insert | column-width truncation |
| 8 | `saveEvent.ts:143-152` | `event_data` rows | query | flatten JSON (EAV) | not atomic with event |
| 9 | `saveEvent.ts:154-166` | `revenue` row | query | if `revenue>0 && currency` | client-supplied revenue |

**Self-check:** which step makes a retried event *not* duplicate the session but *would* duplicate the event? (Step 4 dedupes the session; step 7 uses a fresh `uuid()` so the event can duplicate.)

---

## Trace 3 — Bearer token to authenticated user (auth)

**Scenario:** an authed API call; follow the token to a `user` object.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `useApi.ts:31` | `Authorization: Bearer <t>` | client | attach header | — |
| 2 | `auth.ts:11-14` | token string | server | `getBearerToken` split | malformed header → undefined |
| 3 | `auth.ts:19` | `{ userId, authKey }` | server | `parseSecureToken` (decrypt+verify) | returns null on bad token |
| 4 | `auth.ts:25-33` | `user` | server | `getUser(userId)` or Redis lookup | two paths (stateless/stateful) |
| 5 | `auth.ts:50-52` | `user.isAdmin` | server | role === admin | admin bypass downstream |
| 6 | `auth.ts:54-59` | `Auth` | server | assemble | — |

**Self-check:** at which step does a stateless vs. Redis-backed token diverge, and what's the revocation consequence of each? (Step 4.)

---

## Trace 4 — An error, throw to response (error path)

**Scenario:** the DB is down during collection; follow the failure.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `saveEvent.ts:107` | thrown `Error` | query | Prisma insert rejects | — |
| 2 | `send/route.ts:225` | propagates up (await) | server | unwinds the try | — |
| 3 | `send/route.ts:314-315` | `serializeError(e)` | server | Error → plain object | — |
| 4 | `send/route.ts:318` | log line | server | `console.log(error)` | no metric/alert |
| 5 | `send/route.ts:320` | 500 + `errorObject` | server | `serverError(...)` | **leaks internals to client** |

**Self-check:** what does the *Kafka* failure path do differently at step 2? (It never reaches here — `sendMessage` swallows the error, so the request returns 200. Trace 4 only fires for the *synchronous* insert paths.)

---

## Trace 5 — A funnel report (dynamic SQL)

**Scenario:** a 3-step funnel report; follow parameters into generated SQL. Fill the blanks first.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `schema.ts:171-196` | validated `parameters` | server | Zod funnel schema (2–8 steps) | bounds the generated SQL |
| 2 | `getFunnel.ts:47` | `{ steps, window }` | query | destructure | — |
| 3 | `getFunnel.ts:106-176` | `level1..N` CTEs | query | reduce over steps | reduce bug affects all funnels |
| 4 | `getFunnel.ts:163` | `params[i] = value` | query | step value → bound param | still parameterized |
| 5 | `getFunnel.ts:152-155` | window join | query | `created_at between l.created_at and +window min` | ordering invariant |
| 6 | `getFunnel.ts:350-367` | `FunnelResult[]` | query | compute dropoff/remaining | division by zero if step0=0 |

**Self-check:** what stops the generated SQL from being unbounded in size? (Zod's `.min(2).max(8)` on steps, step 1.)

---

## Trace 6 — A cached website read (cache path)

**Scenario:** collection needs a website; follow the cache-aside read.

| Step | File/line | Value shape | Owner | Transformation | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | `send/route.ts:116` | `websiteId` | server | call `fetchWebsite` | — |
| 2 | `load.ts:9-13` | cache hit/miss | lib | `redis.fetch(key, query, 86400)` | 24h staleness |
| 3 | `redis.ts:85-97` | value or query result | lib | tombstone check, then query | falsy result not cached (`if(data)`) |
| 4 | `load.ts:15` | `null` if deletedAt | lib | soft-delete filter | must not serve deleted |
| 5 | `record/route.ts:56-57` | fresh website | server | **bypasses** cache | correctness over latency |

**Self-check:** contrast steps 1–4 (collection, cached) with step 5 (record, uncached). Why the opposite choice? (Replay's `replayEnabled` must be fresh; a pageview tolerates 24h-stale website metadata.)

---

## Debrief

If you could fill the blank tables (2 and 5) with ≤2 errors each, you can follow data across an unfamiliar codebase — the exact skill a "read this code and explain it" interview round tests. The rows you most often miss are the **risk** column: naming *what could go wrong at each hop* is what separates tracing from understanding. Continue to [`03-fake-code-contrasts.md`](03-fake-code-contrasts.md).
