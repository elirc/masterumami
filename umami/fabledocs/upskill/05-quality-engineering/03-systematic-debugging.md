# Systematic Debugging

The method: **reproduce → narrow → hypothesize → test cheaply → fix root cause → add regression coverage.** The goal is to fix the *cause*, not the symptom, and to leave a test so it can't come back. Five realistic Umami scenarios follow, each ending with the interview version — how to narrate the same debugging aloud in an interview round.

Tools you actually have here: Node/Next server logs, `debug` namespaces (`umami:*` — e.g. `DEBUG=umami:* pnpm dev`), `LOG_QUERY=1` to log SQL ([`prisma.ts:256-260`](../../src/lib/prisma.ts)), browser devtools/Network, and `pnpm test`/Cypress. Commands `__inferred__`.

---

## Scenario 1: "Visitor counts drop to near-zero on the 1st of every month"

**Reproduction:** Compare unique-visitor counts across a month boundary for a stable-traffic site; the count of *returning* visitors resets.

**First question:** Is this a data-loss bug (events missing) or an identity bug (same people counted as new)?

**Narrowing path:**
1. Check whether *event* counts also drop, or only *visitor* counts. If events are steady but visitors reset → identity, not loss.
2. Look at how `sessionId` is derived: [`send/route.ts:143-147`](../../src/app/api/send/route.ts). It hashes in `sessionSalt`.
3. Look at `getSalt` ([`crypto.ts:72-78`](../../src/lib/crypto.ts)): with `SALT_ROTATION=month` (the default), the salt is `startOfMonth(createdAt)`.

**Useful probes:** Log `sessionSalt` for two events from the same client on the 31st and the 1st; they differ.

**Likely root cause:** Not a bug — it's the designed privacy behavior. The monthly salt rotation intentionally makes a returning visitor un-linkable across months, so they count as new. The "fix" depends on intent: if the business wants longer identity windows, set `SALT_ROTATION=day`... no, that's *shorter*. There is no rotation longer than month here, by design.

**Regression test / guard:** Add a test pinning that `getSalt('month', jan31)` !== `getSalt('month', feb1)` so no one "fixes" the reset without understanding it's intentional (Ticket 17).

**Senior lesson:** The hardest bugs are the ones that are working as designed. Before "fixing," confirm whether the behavior is intended by reading the mechanism, not the symptom.

**Interview version:** *"I'd first separate whether events or only visitors dropped — that tells me loss vs. identity. Tracing the id derivation, I'd find a monthly salt rotation, realize it's a deliberate privacy feature, and reframe the ticket from 'bug' to 'product decision about identity window.' The lesson I'd share is: verify intent before fixing."*

---

## Scenario 2: "Some events return 200 but never appear in the dashboard"

**Reproduction:** Send events with a ClickHouse+Kafka deployment while the Kafka broker is briefly unreachable; the tracker gets success responses but the events are missing.

**First question:** Is the loss on the write side (never persisted) or the read side (persisted but not queried)?

**Narrowing path:**
1. Confirm the events aren't in ClickHouse at all → write-side loss.
2. Trace the write: `saveEvent` ClickHouse branch sends via Kafka when enabled ([`saveEvent.ts:259-263`](../../src/queries/sql/events/saveEvent.ts)).
3. Read `sendMessage` ([`kafka.ts:66-91`](../../src/lib/kafka.ts)): it wraps `producer.send` in try/catch and only `console.log`s on failure — returning `undefined`, not throwing.

**Useful probes:** Grep server logs for `KAFKA ERROR`. Their presence, alongside 200 responses to `/api/send`, confirms it.

**Likely root cause:** Fire-and-forget delivery ([Pattern 14](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-14-fire-and-forget-side-effect)). A broker outage is swallowed, so `/api/send` still returns success while the event is dropped. At-most-once semantics.

**Root-cause fix (not symptom):** Don't just add a retry loop. The real fix is a delivery guarantee: buffer to a durable outbox and relay, or propagate the send failure so the client can retry. Because `sessionId`/`event_id` are deterministic/unique, at-least-once retries are idempotent ([Pattern 9](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-9-idempotent-insert-on-conflict-do-nothing)).

**Regression test:** A unit test that stubs `producer.send` to throw and asserts the chosen new behavior (error surfaced or outbox row written).

**Senior lesson:** A 200 that hides data loss is worse than a 500. Silent catch blocks are where reliability bugs hide; make failure visible before making it rare.

**Interview version:** *"Success-but-missing points at a swallowed error on the write path. I'd grep for the log line, find a fire-and-forget Kafka send that catches and drops, and recognize at-most-once delivery. I wouldn't slap on a retry — I'd argue for an outbox, and note the deterministic ids make retries safe from duplicates."*

---

## Scenario 3: "A user can see another user's website metrics"

**Reproduction:** As user A, call `GET /api/websites/<B's id>/<some-endpoint>` with A's valid token; data comes back.

**First question:** Is authentication failing (A isn't really authed) or authorization (A is authed but not checked against the resource)?

**Narrowing path:**
1. Confirm A's token is valid and resolves to A → authentication is fine.
2. The bug is authorization. Compare the leaking route against a correct one: correct routes call `canViewWebsite(auth, websiteId)` right after `parseRequest` ([`realtime/[websiteId]/route.ts:20`](../../src/app/api/realtime/[websiteId]/route.ts), [`websites/[websiteId]/route.ts:28`](../../src/app/api/websites/[websiteId]/route.ts)).
3. The leaking route is missing that `can*` call, or calls it against the wrong id.

**Useful probes:** Grep the route for `canView`/`canUpdate`. Absence is the smoking gun.

**Likely root cause:** A missing per-handler authorization check — the classic IDOR. `parseRequest` authenticates but deliberately does *not* authorize (it can't know the resource, [`request.ts:44-50`](../../src/lib/request.ts)).

**Root-cause fix:** Add the correct `canViewWebsite(auth, websiteId)` guard returning `unauthorized()` on failure — matching the pattern in sibling routes.

**Regression test:** A Cypress/API test asserting A gets 401 for B's website (extend `cypress/e2e/api-website.cy.ts`).

**Senior lesson:** Authentication ≠ authorization. Because authz is per-handler here, the systemic defense is a review checklist (or a wrapper) ensuring every id-bearing route authorizes. Fixing the one route treats the symptom; adding the checklist treats the class.

**Interview version:** *"I'd confirm the token is valid, which localizes it to authorization. Comparing the route to its siblings, I'd see a missing `canViewWebsite` call — an IDOR. I'd fix the route and then propose a guardrail so the whole class can't recur, because per-handler authz is easy to forget."*

---

## Scenario 4: "The sessions list is slow and sometimes times out for big sites"

**Reproduction:** Open the sessions view for a high-traffic website over a wide date range; the request is slow.

**First question:** Is it slow in the DB (query) or in the app (serialization/render)?

**Narrowing path:**
1. Set `LOG_QUERY=1` and read the emitted SQL for `getWebsiteSessions` ([`getWebsiteSessions.ts:33-77`](../../src/queries/sql/sessions/getWebsiteSessions.ts)).
2. Check the `where`/`join`: it filters `website_event.website_id` + date, joins `session`, groups by 11 columns. Confirm the composite index `(website_id, created_at, ...)` is being used ([`schema.prisma:139-148`](../../prisma/schema.prisma)).
3. Note `pagedRawQuery` runs a `count(*)` over the same subquery *and* the page query ([`prisma.ts:325-329`](../../src/lib/prisma.ts)) — two scans per request.

**Useful probes:** `EXPLAIN ANALYZE` the logged SQL against a realistic dataset; watch for a sequential scan or a huge sort.

**Likely root cause:** The unbounded date range makes the grouped scan large; the extra `count(*)` doubles the work; and `pageSize` has no upper bound ([`schema.ts:79`](../../src/lib/schema.ts)), so a client can request an enormous page.

**Root-cause fix:** Bound `pageSize` (Ticket 5); consider whether the total `count` is needed for every page (cursor pagination avoids it); ensure the date range has a sane default cap. Measure before and after — don't guess.

**Regression test:** A performance assertion is hard in unit tests; instead add the `pageSize` bound test and document the query cost.

**Senior lesson:** "Measure first." The instinct to "add an index" is often wrong — the indexes exist; the cost was an unbounded range plus a redundant count. Read the actual query plan before changing anything.

**Interview version:** *"I'd log the SQL and EXPLAIN it rather than guessing. I'd likely find the indexes are fine but the date range is unbounded and pagination does a second full count. The fix is bounding inputs and reconsidering the count, not adding indexes. My headline: measure the plan before you optimize."*

---

## Scenario 5: "Disabling session replay doesn't take effect for up to a day"

**Reproduction:** Toggle `replayEnabled=false` on a website; the collection path keeps behaving as if replay were on for a while, but the record endpoint stops immediately.

**First question:** Why does one path react instantly and another lag?

**Narrowing path:**
1. Find where each path reads the website. Collection uses `fetchWebsite` (Redis-cached 24h, [`load.ts:6-13`](../../src/lib/load.ts)). Record uses `getWebsite` directly ([`record/route.ts:56-57`](../../src/app/api/record/route.ts)).
2. The record path even has a comment: "Query directly to avoid stale Redis cache for recordingEnabled."

**Useful probes:** Check the Redis TTL for the `website:<id>` key (86400s).

**Likely root cause:** The cached website (24h TTL) serves a stale `replayEnabled` on any path that reads through `fetchWebsite`. The record path deliberately bypasses the cache; other paths don't.

**Root-cause fix:** On the settings-update handler, *invalidate* the cache key (`redis.remove('website:<id>', soft)`) when `replayEnabled`/`replayConfig` change — so the cache reflects the change without waiting for TTL. Read [`websites/[websiteId]/route.ts:71-76`](../../src/app/api/websites/[websiteId]/route.ts) to see where the update happens.

**Regression test:** Assert that updating a website removes/refreshes its cache entry.

**Senior lesson:** Caching is easy to add and hard to invalidate. When a value can change, the write path must invalidate the cache, or the read path must bypass it — pick one deliberately per value. The inconsistency here (one path bypasses, others don't) is the actual defect.

**Interview version:** *"Two paths, two behaviors means two different reads. I'd find one uses a 24h Redis cache and one reads fresh, with a comment admitting the staleness. The real fix is cache invalidation on update, not bypassing the cache everywhere. It's the classic 'there are only two hard problems' lesson, grounded in a real comment in the code."*

---

## Applying the method

For any bug: **never fix at the first plausible line.** Reproduce it, split the system in half (which side is it on?), form one hypothesis, test it cheaply (a log, a `LOG_QUERY`, an EXPLAIN), and only then fix — at the cause. Add the regression test *before* you close it. The debugging interview round (`08-interview-prep/05-debugging-and-code-review-rounds.md`) turns three of these into timed simulations with interviewer follow-ups.
