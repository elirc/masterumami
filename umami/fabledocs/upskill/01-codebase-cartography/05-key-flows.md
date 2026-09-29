# Key Flows

Six end-to-end flows, traced with anchors. These are the backbone of the whole curriculum — later modules reference them by name rather than re-tracing. They rotate across the surfaces the prompt asks for: a client/collection flow, a server write flow, a persistence flow, a read/authz flow, an async/replay flow, and an auth flow.

Flow index:
1. [Collection — the write hot path](#flow-1-collection--the-write-hot-path) (client → API → DB)
2. [Reading a metric — the authorized read path](#flow-2-reading-a-metric--the-authorized-read-path) (UI → API → SQL)
3. [Login and the two token strategies](#flow-3-login-and-the-two-token-strategies) (auth boundary)
4. [Pixel & link tracking — handler composition](#flow-4-pixel--link-tracking--handler-composition)
5. [Session replay ingestion](#flow-5-session-replay-ingestion) (async-ish / large payload)
6. [Sharing a dashboard publicly](#flow-6-sharing-a-dashboard-publicly) (a second auth path)

---

## Flow 1: Collection — the write hot path

**Why this flow matters:** This is the product. Every design decision — cookieless identity, unauthenticated ingest, dual persistence, a signed client cache — lives here. If you understand only one flow, understand this one.

**Open these files first:**
- [`src/tracker/index.js`](../../src/tracker/index.js) — the browser script (`getPayload` `:64`, `send` `:163`, `track` `:207`).
- [`src/app/api/send/route.ts`](../../src/app/api/send/route.ts) — the server handler (`POST` `:66`).
- [`src/lib/crypto.ts`](../../src/lib/crypto.ts) — identity derivation (`uuid` `:60`, `getSalt` `:72`).
- [`src/lib/db.ts`](../../src/lib/db.ts) — dispatch (`runQuery` `:22`).
- [`src/queries/sql/events/saveEvent.ts`](../../src/queries/sql/events/saveEvent.ts) — the write.

**Trace:**

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Browser | `src/tracker/index.js:64` | Build payload from `location`, `document.title`, `referrer`, screen size. No cookie, no id. | `{ website, url, referrer, title, screen, language, tag }` | Payload is fully client-controlled; server must not trust it. |
| 2 | Browser | `src/tracker/index.js:175` | `fetch(endpoint, { keepalive: true, method:'POST', body })`, attaching `x-umami-cache` header if present. | JSON `{ type, payload }` | `keepalive` lets the send survive page unload — but caps body size. |
| 3 | Server | `src/app/api/send/route.ts:68` | `parseRequest(request, schema, { skipAuth: true })` — Zod-validate, **no auth**. | Validated `payload` | Open endpoint; validation is the *only* gate. |
| 4 | Server | `src/app/api/send/route.ts:104-112` | If `x-umami-cache` present, `parseToken` it. A valid token carries `{ websiteId, sessionId, visitId, iat }` and lets the handler **skip the website lookup and session insert**. | `Cache` object | Token is signed (`secret()`), so not forgeable — but it is client-held state. |
| 5 | Server | `src/app/api/send/route.ts:116` | Cache miss → `fetchWebsite(websiteId)` (Redis-cached, `src/lib/load.ts:6`). 404 if not found. | `Website \| null` | Cache-aside; a soft-deleted website returns null (`load.ts:15`). |
| 6 | Server | `src/app/api/send/route.ts:125` | `getClientInfo` resolves IP, UA → browser/os/device, and geo (headers → MaxMind). | `{ ip, userAgent, browser, os, country, region, city, device }` | Geo from proxy headers is spoofable; MaxMind is the fallback. |
| 7 | Server | `src/app/api/send/route.ts:131-138` | Bot check (`isbot`) returns `{beep:'boop'}`; blocked IP → `403`. | early return | Bots are silently accepted-then-ignored (returns 200-ish JSON), not errored. |
| 8 | Server | `src/app/api/send/route.ts:147` | **Identity**: `sessionId = uuid(sourceId, ip, userAgent, sessionSalt)` — a deterministic v5 UUID (a hash), not random. | `sessionId: string` | Same visitor+day+salt → same id. No PII stored. |
| 9 | Server | `src/app/api/send/route.ts:150-165` | On the relational path only, `createSession(...)` with `on conflict do nothing` (`createSession.ts:39`). | insert | Idempotent insert — safe to repeat. ClickHouse path skips this (session is denormalized into the event). |
| 10 | Server | `src/app/api/send/route.ts:168-175` | Derive `visitId`; expire the visit after 30 min of inactivity by re-hashing. | `visitId: string` | Visit = session bucketed into 30-min windows. |
| 11 | Server | `src/app/api/send/route.ts:177-271` | For `type==='event'`, parse URL/referrer, extract UTM + click ids, then `saveEvent(...)`. | `SaveEventArgs` | URL parsing uses a synthetic base when hostname missing (`:178`). |
| 12 | Server | `src/lib/db.ts:22` | `runQuery` routes to Prisma **or** ClickHouse **or** Kafka based on `CLICKHOUSE_URL` / `DATABASE_URL`. | dispatch | Exactly one store is written — never both. |
| 13 | Server | `src/queries/sql/events/saveEvent.ts:107-167` | Relational: `prisma.client.websiteEvent.create`, then conditional `saveEventData` and `saveRevenue`. | rows | Sequential awaits, no wrapping transaction across event+data+revenue. |
| 14 | Server | `src/app/api/send/route.ts:311-313` | Sign a fresh cache token and return `{ cache, sessionId, visitId }`. Tracker stores `cache` for step 4 next time. | JSON | Response teaches the client how to skip work next request. |

**Validation and authorization:** Validation is the Zod `schema` at `src/app/api/send/route.ts:24-64`, including a `.refine` enforcing exactly one of `website`/`link`/`pixel` (`:53-63`). Authorization is intentionally absent (`skipAuth: true`, `:68`) — collection is public by design. The *only* server-side trust boundary is the schema plus bot/IP filtering.

**Persistence and side effects:** `saveEvent` (`saveEvent.ts:65`) → optional `saveEventData` (JSON flattened to typed rows, `saveEventData.ts:27`) → optional `saveRevenue` when `eventData.revenue > 0` (`saveEvent.ts:154-166`). On the ClickHouse path, writes go through Kafka if enabled (`saveEvent.ts:259-263`).

**Tests that cover it:** No direct unit test of the `POST /api/send` handler was found in `src/` (searched; `saveEvent`/route have no `.test.ts`). Related pure helpers are tested: `src/lib/detect.test.ts`, `src/lib/format.test.ts`. This is a genuine test-coverage gap and a ticket in [`06-contribution-practice/01-good-first-tickets.md`](../06-contribution-practice/01-good-first-tickets.md).

**What juniors usually miss:** That there is no cookie and no client-sent id — identity is *reconstructed server-side by hashing*. And that the `x-umami-cache` token is a performance optimization, not an auth token.

**What seniors notice:** The cache token makes the client hold server state, so a website's cached `deletedAt`/`replayEnabled` can be stale (which is exactly why `/api/record` bypasses the cache — Flow 5). The Kafka path swallows send errors (`src/lib/kafka.ts:87-90`), so a broker outage silently drops events with a 200 response. There is no application-level rate limit on this open endpoint (`redis.rateLimit` exists but is uncalled — `src/lib/redis.ts:72`).

**Interview angle:** This flow *is* the "design a web analytics / high-write ingestion service" system-design question. See [`08-interview-prep/04-system-design-from-this-repo.md`](../08-interview-prep/04-system-design-from-this-repo.md).

**Drill:** Trace what happens on the *second* pageview from the same browser within 5 minutes. Which of steps 5, 9 are skipped, and why? Write the answer, then check against steps 4 and 10.

**Self-grade:** *Basic* — you can name the endpoint and the DB write. *Solid* — you can explain the cache token and the hashed identity and point at both anchors. *Strong* — you can explain why bot traffic returns 200, why there's no transaction across event/data/revenue, and one consistency risk of the client-held cache token.

---

## Flow 2: Reading a metric — the authorized read path

**Why this flow matters:** It's the mirror image of Flow 1 and the template every dashboard read follows: hook → `useApi` → `parseRequest` → permission check → parameterized SQL → JSON. Learn it once, recognize it 60 times.

**Open these files first:**
- [`src/components/hooks/queries/useWebsiteMetricsQuery.ts`](../../src/components/hooks/queries/useWebsiteMetricsQuery.ts) (`:12`)
- [`src/components/hooks/useApi.ts`](../../src/components/hooks/useApi.ts) (`:20`)
- [`src/lib/request.ts`](../../src/lib/request.ts) (`parseRequest` `:12`)
- [`src/permissions/website.ts`](../../src/permissions/website.ts) (`canViewWebsite` `:7`)
- [`src/lib/prisma.ts`](../../src/lib/prisma.ts) (`rawQuery` `:255`)

**Trace:**

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | UI | `useWebsiteMetricsQuery.ts:21` | React Query call; `queryKey` includes websiteId + date range + filters. | `queryKey` array | Wrong key granularity → stale or over-fetched data. |
| 2 | UI | `useApi.ts:45-50` | `get(url, params, headers)` attaches `Authorization: Bearer <token>` from localStorage and share headers if on a `/share` path. | Request | Token in localStorage is readable by any XSS on the page. |
| 3 | API | `src/app/api/websites/[websiteId]/metrics/route.ts` | Route handler runs `parseRequest(request, schema)`. | — | — |
| 4 | API | `request.ts:23-42` | Zod validates the query. On GET, unknown suffixed filter keys (`browser1`, `os2`) are re-added after Zod strips them (`:33-38`). | `query` | The re-add step is a deliberate escape hatch around Zod's stripping. |
| 5 | API | `request.ts:44-50` → `auth.ts:17` | `checkAuth`: parse bearer (secure token) or share token; look up the user; set `isAdmin`. | `Auth \| null` | 401 if neither user nor valid share token. |
| 6 | API | `permissions/website.ts:7` | `canViewWebsite(auth, websiteId)`: admin bypass → share-token match → entity ownership → team membership. | boolean | **This is the IDOR guard.** Skipping it would let any user read any website. |
| 7 | API | `request.ts:111` | `getQueryFilters` builds date range, filters, segment/cohort expansion. | `QueryFilters` | Segment params are loaded from the DB and merged in. |
| 8 | Query | `src/lib/prisma.ts:232-253` | `parseFilters` turns the filter object into SQL fragments + a param map. | `{ filterQuery, queryParams, ... }` | Filters become `= ANY($n)` etc., always parameterized. |
| 9 | Query | `src/lib/prisma.ts:255-282` | `rawQuery` replaces `{{name::type}}` with `$1,$2,...` and calls `$queryRawUnsafe(query, ...params)`. | rows | "Unsafe" refers to the raw string; **values are still bound**, not concatenated. |
| 10 | API | route | Result serialized to JSON; `BigInt.prototype.toJSON` patched to `Number` (`src/lib/db.ts:8-10`). | JSON | BigInt would otherwise throw on `JSON.stringify`. |

**Validation and authorization:** Validation = the route's Zod schema via `parseRequest`. Authorization = `canViewWebsite` (`permissions/website.ts:7`). Note the two-phase shape used by nearly every read route: *authenticate in `parseRequest`, then authorize with an explicit `can*` call in the handler.* They are deliberately separate concerns.

**Persistence and side effects:** Read-only. If `DATABASE_REPLICA_URL` is set, `rawQuery` routes to a read replica (`prisma.ts:278-281`) — a scaling lever worth noting for the system-design interview.

**Tests that cover it:** e2e coverage exists at the API level in `cypress/e2e/api-website.cy.ts`. No unit test isolates `canViewWebsite`; that's a good-first-ticket.

**What juniors usually miss:** That `$queryRawUnsafe` is *not* an injection hole here — the `{{param}}` → `$n` rewrite (`prisma.ts:268-276`) keeps values bound. The "unsafe" is about the query *text* being dynamic, which is where the real (latent) risk lives — see the `orderBy` note in Flow-adjacent risk below.

**What seniors notice:** Authentication and authorization are separate steps and both are required; forgetting the `can*` call is the classic IDOR. The suffixed-filter re-add (`request.ts:33-38`) is a smell worth a code comment. `pagedRawQuery` interpolates `orderBy` directly (`prisma.ts:318-323`) — safe today only because no read route feeds it user input.

**Interview angle:** "How do you prevent one user from reading another's data?" → point at the authenticate-then-authorize split and `canViewWebsite`. See [`08-interview-prep/03-api-and-data-modeling-questions.md`](../08-interview-prep/03-api-and-data-modeling-questions.md).

**Drill:** Delete the `canViewWebsite` check mentally. Write the exact request an attacker would send to read another tenant's metrics, and name the single line that stops it.

**Self-grade:** *Basic* — you can trace hook → route → SQL. *Solid* — you distinguish authentication from authorization and cite both anchors. *Strong* — you can explain why `$queryRawUnsafe` is safe here but `orderBy` interpolation is a latent risk.

---

## Flow 3: Login and the two token strategies

**Why this flow matters:** It shows a real, subtle decision: Umami issues **either** a stateless encrypted JWT **or** a Redis-backed opaque key, depending on whether Redis is configured. That "stateless vs. server-side session" tradeoff is a top interview topic.

**Open these files first:**
- [`src/app/api/auth/login/route.ts`](../../src/app/api/auth/login/route.ts) (`:12`)
- [`src/lib/auth.ts`](../../src/lib/auth.ts) (`saveAuth` `:62`, `checkAuth` `:17`)
- [`src/lib/jwt.ts`](../../src/lib/jwt.ts) (`createSecureToken` `:16`)
- [`src/lib/password.ts`](../../src/lib/password.ts)

**Trace:**

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API | `login/route.ts:26` | Look up user by username (with password hash). | `User` | — |
| 2 | API | `login/route.ts:28` | `checkPassword(password, user.password)` (bcrypt compare). Same error for bad user or bad password. | boolean | Uniform error avoids username enumeration. |
| 3 | API | `login/route.ts:36-40` | **If Redis**: `saveAuth({userId, role})` stores in Redis under `auth:<random>` and returns an encrypted token wrapping the key. **Else**: `createSecureToken({userId, role})` — a self-contained encrypted JWT. | `token: string` | Stateless token can't be revoked server-side; Redis token can. |
| 4 | Client | `src/app/login/LoginForm.tsx:25-28` | On success, `setClientAuthToken(token)` (localStorage) and `setUser`. | — | localStorage token → XSS-readable. |
| 5 | API (later) | `auth.ts:17-33` | Every authed request: `parseSecureToken` the bearer; if it carries a `userId`, load the user; if it carries an `authKey` and Redis is on, look up the session. | `Auth` | Two decode paths, one per issuance strategy. |

**Validation and authorization:** Credentials validated by Zod (`login/route.ts:13-16`); password by bcrypt (`checkPassword`). No authorization — anyone may attempt login.

**Persistence and side effects:** With Redis, a session row is written (`saveAuth`, `auth.ts:62-74`) with optional TTL. Without Redis, nothing is persisted — the token *is* the session.

**Tests that cover it:** `cypress/e2e/login.cy.ts` covers success, blank inputs, and wrong password against the real UI.

**What juniors usually miss:** That `createSecureToken` **encrypts** the JWT (`jwt.ts:16-18`), not just signs it — so the token body isn't readable by the client, unlike a normal JWT. And that the same endpoint can produce two structurally different tokens.

**What seniors notice:** Stateless tokens can't be revoked before expiry — a logout only clears localStorage. The Redis path buys revocation and TTL at the cost of a lookup per request. Choosing between them is the classic "sessions vs. JWT" tradeoff, and Umami picked *both, by config*.

**Interview angle:** "JWT vs. server-side sessions — when would you use each?" You can answer with a codebase that does both. See [`08-interview-prep/01-js-ts-node-deep-dive.md`](../08-interview-prep/01-js-ts-node-deep-dive.md).

**Drill:** A user reports "I logged out but my old link still works." Under which token strategy is that expected, and what would you change to fix it?

**Self-grade:** *Basic* — login checks a password and returns a token. *Solid* — you can explain the Redis-vs-stateless branch and its revocation consequence. *Strong* — you can argue when encryption (not just signing) of the token matters and cite `jwt.ts:16`.

---

## Flow 4: Pixel & link tracking — handler composition

**Why this flow matters:** It's a clean example of **composition over duplication**: the pixel and link routes don't re-implement collection — they build a synthetic `Request` and call the `send` handler directly. Great "how do you avoid duplication across endpoints?" material.

**Open these files first:**
- [`src/app/(collect)/p/[slug]/route.ts`](../../src/app/(collect)/p/[slug]/route.ts) — pixel (`:12`)
- [`src/app/(collect)/q/[slug]/route.ts`](../../src/app/(collect)/q/[slug]/route.ts) — link (`:10`)

**Trace:**

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API | `p/[slug]/route.ts:17-43` | Resolve pixel by slug, Redis-cached 24h. 404 if missing. | `Pixel` | Cache-aside; slug is public. |
| 2 | API | `p/[slug]/route.ts:45-58` | Build `{ type:'event', payload:{ pixel: id, url, referrer } }` and a new POST `Request`. | synthetic Request | Reuses inbound headers (IP, UA). |
| 3 | API | `p/[slug]/route.ts:60` | `await POST(req)` — calls the *same* `send` handler as Flow 1. | — | Tight coupling to `send`'s signature. |
| 4 | API | `p/[slug]/route.ts:62-70` | Return a 1×1 transparent GIF with no-cache headers. | image/gif | Pixel works even where JS is blocked (email opens). |
| 4′ | API | `q/[slug]/route.ts:60` | Link variant instead returns `NextResponse.redirect(link.url)`. | 302 | Tracks the click, then forwards. |

**Validation and authorization:** None at these routes beyond slug existence; the reused `send` handler applies the Zod schema and bot/IP checks.

**Persistence and side effects:** Same as Flow 1 (an event write), plus the GIF/redirect response.

**What juniors usually miss:** That calling `POST(req)` directly is calling another route handler as a function — legal because Next.js route handlers are just exported functions.

**What seniors notice:** The composition couples three routes to `send`'s exact request contract; a change to `send`'s schema silently affects pixel/link. The `batch` route (`src/app/api/batch/route.ts:36`) uses the same trick per item. A shared internal function would decouple them — a legitimate refactor kata.

**Interview angle:** "How do you share logic across endpoints without duplication, and what's the cost?" See the pattern card "Handler-as-function composition" in [`03-architecture-and-patterns/05-pattern-catalog.md`](../03-architecture-and-patterns/05-pattern-catalog.md).

**Drill:** List every route that would break if `send`'s Zod schema renamed `payload.website` to `payload.websiteId`. (Hint: grep for `send.POST` and `POST(req)`.)

**Self-grade:** *Basic* — pixel returns a GIF, link redirects. *Solid* — you see both reuse the `send` handler. *Strong* — you can name the coupling risk and propose the extract-shared-function refactor.

---

## Flow 5: Session replay ingestion

**Why this flow matters:** It's the closest thing to an async/large-payload flow: the recorder batches rrweb events and flushes them; the server stores compressed chunks. It also demonstrates *why* you sometimes bypass a cache.

**Open these files first:**
- [`src/recorder/index.js`](../../src/recorder/index.js) (`:1`)
- [`src/app/api/record/route.ts`](../../src/app/api/record/route.ts) (`:27`)
- [`src/queries/sql/replays/saveRecording.ts`](../../src/queries/sql/replays/saveRecording.ts)

**Trace:**

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | Browser | `src/recorder/index.js:13-22` | Read config; **sample-rate gate** — `Math.random() > sampleRate` aborts recording entirely. | — | Sampling keeps volume down; also non-deterministic. |
| 2 | Browser | `src/recorder/index.js:28` | rrweb `record()` buffers DOM mutation events; flush at 100 events or 10s. | event array | Large payloads; capped at 200/request server-side. |
| 3 | API | `record/route.ts:29-35` | `parseRequest(..., {skipAuth:true})`, Zod caps `events` at 200 (`:23`). | `{ website, events, timestamp }` | Unauthenticated, like collection. |
| 4 | API | `record/route.ts:42-54` | Require a valid `x-umami-cache` token to recover `sessionId`/`visitId`. | `Cache` | No session token → 400. Replay can't start a session. |
| 5 | API | `record/route.ts:56-65` | `getWebsite(websiteId)` **directly** (not the Redis-cached `fetchWebsite`) to read a fresh `replayEnabled`. | `Website` | Deliberate cache bypass — a comment says so (`:56`). |
| 6 | API | `record/route.ts:67-77` | CLOUD_MODE billing gate: business subscription required. | early 403 | Feature-gated in cloud. |
| 7 | API | `record/route.ts:91-114` | Compute `startedAt`/`endedAt` from event timestamps; `saveRecording(...)` stores the chunk as bytes. | `SessionReplay` row | Timestamp math tolerates missing/invalid values. |

**Validation and authorization:** Zod schema (`record/route.ts:18-25`); no user auth, but a valid cache token is required (a *session* proof, not a *user* proof). Billing gate in cloud mode.

**Persistence and side effects:** One `SessionReplay` row per flushed chunk (`prisma/schema.prisma:362`), storing `events` as `Bytes`. Chunks are ordered by a timestamp-derived `chunkIndex`.

**What juniors usually miss:** That recording is sampled on the client, so most sessions are never recorded — and that the endpoint needs the cache token from Flow 1 to attach the replay to a session.

**What seniors notice:** The explicit cache bypass (`:56-57`) is a correctness-over-latency choice: a freshly toggled `replayEnabled=false` must take effect immediately, so the cached website (24h TTL) can't be trusted here. That contrast with Flow 1's `fetchWebsite` is the whole lesson about *when caching is and isn't safe*.

**Interview angle:** "When would you deliberately skip your cache?" → point at `record/route.ts:56`. See [`05-quality-engineering/04-performance-thinking.md`](../05-quality-engineering/04-performance-thinking.md).

**Drill:** The team reports that disabling replay "doesn't take effect for up to a day" on the collection path but is instant on the record path. Explain the discrepancy using the two website-fetch calls.

**Self-grade:** *Basic* — replay stores DOM events. *Solid* — you see the cache bypass and can say why. *Strong* — you can contrast the TTL tradeoff against Flow 1 and identify sampling as a data-completeness caveat.

---

## Flow 6: Sharing a dashboard publicly

**Why this flow matters:** It's a *second* authorization path that runs entirely without a user account, using a share token. Multi-path authz ("logged-in user OR share link") is exactly the kind of subtlety interviewers and real bugs live in.

**Open these files first:**
- [`src/app/api/share/[slug]/route.ts`](../../src/app/api/share/[slug]/route.ts) (`:47`)
- [`src/lib/auth.ts`](../../src/lib/auth.ts) (`parseShareToken` `:80`, `checkAuth` share branch `:42-48`)
- [`src/permissions/website.ts`](../../src/permissions/website.ts) (share-token match `:12-21`)

**Trace:**

| Step | Owner | File | What happens | Data shape | Risk |
| --- | --- | --- | --- | --- | --- |
| 1 | API | `share/[slug]/route.ts:50-52` | Resolve share by public slug; 404 if missing. | `Share` | Slug is the only secret. |
| 2 | API | `share/[slug]/route.ts:64-92` | Depending on `shareType` (board/website/pixel/link), gather the entity ids the share grants. | `data` incl. ids | A board share expands to many entity ids. |
| 3 | API | `share/[slug]/route.ts:94` | `createToken(data, secret())` — a signed token embedding the granted ids. | share token | Signed, so scope can't be widened by the client. |
| 4 | Client | `useApi.ts:25-28` | On `/share` paths, send the token via `x-umami-share-token` **and** `x-umami-share-context` headers. | headers | Context header required (see step 6). |
| 5 | API | `auth.ts:20,42-48` | `checkAuth` parses the share token; if there's no user, it requires the share-*context* header to be present. | `Auth{ shareToken }` | Context header gates share-token use to share pages. |
| 6 | API | `permissions/website.ts:12-21` | `canViewWebsite` returns true if the requested id is in the share token's granted ids. | boolean | Authz without a user account. |

**Validation and authorization:** The share token *is* the authorization; `canViewWebsite` checks membership of the requested id in the token's id lists (`permissions/website.ts:12-21`). The `SHARE_CONTEXT_HEADER` requirement (`auth.ts:43-47`) is a guard so a leaked share token can't be used as a general API credential.

**Persistence and side effects:** Read-only. A `Share` row is created earlier when a user enables sharing (see `websites/[websiteId]/route.ts:82-90`).

**What juniors usually miss:** That there are two independent ways to pass `canViewWebsite` — a real user, or a share token — and both must be kept correct as the code evolves.

**What seniors notice:** The share token embeds granted ids at creation time; if a board's membership changes later, the old token may over- or under-grant until reissued. The context-header guard is a small but important blast-radius reduction.

**Interview angle:** "How do you implement public share links safely?" → signed, scoped token + context guard. See [`03-architecture-and-patterns/03-validation-auth-and-permissions.md`](../03-architecture-and-patterns/03-validation-auth-and-permissions.md).

**Drill:** An attacker copies a valid `x-umami-share-token` and replays it against `/api/websites/<id>/export` (not a share page). Which line stops them, and what happens if you remove it?

**Self-grade:** *Basic* — a slug maps to a shared dashboard. *Solid* — you can explain the signed, scoped token. *Strong* — you can explain the context-header guard and the stale-scope risk of embedding ids at creation time.

---

## How these flows are reused

- Annotation drills in [`04-code-reading-gym/01-annotation-drills.md`](../04-code-reading-gym/01-annotation-drills.md) pull excerpts from Flows 1, 2, 5.
- Trace tables in [`04-code-reading-gym/02-trace-tables.md`](../04-code-reading-gym/02-trace-tables.md) extend Flows 1 and 2.
- The system-design interview (`08-.../04`) reverse-engineers Flow 1.
- The risk register (`09-reference/risk-register.md`) formalizes the "what seniors notice" rows above.
