# Side Effects, Async & Reliability

A **side effect** is anything a function does beyond returning a value: a DB write, a network call, a cache mutation, a log. Reliability is about what happens when those side effects fail. This page maps every side effect in Umami and grades its reliability posture, teaching idempotency, retries, at-least/at-most-once, and where the gaps are.

## Every side effect (the inventory)

| Side effect | Where | Failure handling | Guarantee |
| --- | --- | --- | --- |
| Insert event (Postgres) | `saveEvent.ts:107` | Throws → caught by route try/catch → 500 | At-least-once (client may retry) |
| Insert event (ClickHouse direct) | `saveEvent.ts:262` | Throws → 500 | At-least-once |
| Publish event (Kafka) | `kafka.ts:73` | **Swallowed** (`console.log`) → 200 | **At-most-once (silent loss)** |
| Insert session | `createSession.ts` | `on conflict do nothing` | Idempotent |
| Insert event_data | `saveEventData.ts` | Throws → 500 | At-least-once, not atomic with event |
| Insert revenue | `saveRevenue.ts` | Throws → 500 | At-least-once, not atomic with event |
| Save replay chunk | `record/route.ts:105` | Throws → 500 | At-least-once |
| Redis cache set | `redis.ts:46` | Connect errors logged | Best-effort (cache, so tolerable) |
| Geo DB lookup | `detect.ts:103-124` | Falls back to headers/undefined | Degrades gracefully |
| Issue tokens | `login`, `send`, `share` | Synchronous, in-request | Deterministic |

## Idempotency (the reliability superpower here)

**Idempotent** = doing it twice has the same effect as once. Umami earns idempotency through **deterministic ids**:

- `sessionId`/`visitId`/`event`-derived ids are hashes of their inputs ([`crypto.ts:60-66`](../../src/lib/crypto.ts)), so a retried write targets the same row.
- `createSession` uses `ON CONFLICT DO NOTHING` ([`createSession.ts:39`](../../src/queries/sql/sessions/createSession.ts)) — a duplicate session insert is a no-op.

This is what makes at-least-once delivery *safe*: if a client retries a failed `/api/send`, the session won't duplicate. The transferable rule: **at-least-once + idempotent key = effectively-once**, which is the standard pattern for reliable messaging. (Note: the *event* row itself uses a fresh `uuid()` per call, `saveEvent.ts:105`, so a retried event *can* duplicate — sessions are idempotent, individual events are not. Worth knowing precisely.)

## The reliability gap: fire-and-forget Kafka

The one place reliability is genuinely weak. `sendMessage` catches and swallows all errors ([`kafka.ts:87-90`](../../src/lib/kafka.ts)), returning `undefined`. So on the ClickHouse+Kafka path, a broker outage means:

- The event is dropped.
- `/api/send` still returns 200 (the caller thinks it succeeded).
- Nothing retries; there's no dead-letter queue or outbox.

This is **at-most-once** delivery for the product's core data. It's the canonical "fire-and-forget" pitfall ([Pattern 14](05-pattern-catalog.md#pattern-14-fire-and-forget-side-effect)). The stronger designs, in order of effort:

1. **Propagate the error** so the client (tracker) can retry — but the tracker's `fetch` catch is also a no-op ([`tracker/index.js:192-194`](../../src/tracker/index.js)), so this alone wouldn't help without tracker changes.
2. **Outbox pattern**: write the event to a durable local table in the same transaction as accepting it, then a relay ships it to Kafka with retries. Guarantees at-least-once; duplicates handled by idempotent keys.
3. **Acked producer + local buffer**: raise `acks`, buffer on failure, retry with backoff.

Being able to walk this from "the bug" to "the fix" to "the new problem the fix creates (duplicates → handled by idempotency)" is a strong systems answer.

## Atomicity: the missing transaction

Event + event_data + revenue are three separate awaits with **no wrapping transaction** ([`saveEvent.ts:143-167`](../../src/queries/sql/events/saveEvent.ts)). If the process dies after the event insert but before revenue, you get an event with no revenue row. For analytics this is tolerable (counts are approximate anyway); the lesson is to *recognize* the missing atomicity and judge whether the domain tolerates it. For money that would be unacceptable — you'd wrap them in `prisma.transaction()` ([`prisma.ts:360`](../../src/lib/prisma.ts)).

## Timeouts, backpressure, retries

- **Kafka timeouts:** `CONNECT_TIMEOUT=5000`, `SEND_TIMEOUT=3000`, `acks=1` ([`kafka.ts:8-10`](../../src/lib/kafka.ts)) — bounded waits so a slow broker doesn't hang requests.
- **Backpressure:** the record endpoint caps events per request at 200 ([`record/route.ts:23`](../../src/app/api/record/route.ts)); the tracker flushes replay at 100 events/10s ([`recorder/index.js:30-31`](../../src/recorder/index.js)). These bound payload size — a crude but real backpressure.
- **No retry/rate-limit layer:** there's no application retry on writes and no rate limiting on ingest (`redis.rateLimit` exists but is uncalled, [`redis.ts:72`](../../src/lib/redis.ts)). Both are "infra is expected to handle it" assumptions worth flagging.

## Failure visibility ("how would I know it broke?")

- Collection errors are `console.log`'d as serialized errors ([`send/route.ts:314-318`](../../src/app/api/send/route.ts)) — but a Kafka drop logs a `KAFKA ERROR` line and *nothing else surfaces*. There are no metrics/alerts in-repo.
- The `debug` library gives namespaced logs (`umami:*`), off by default.
- Health is a shallow `/api/heartbeat` used by the container healthcheck ([`docker-compose.yml`](../../docker-compose.yml)).

So the honest answer to "how would you know events are being dropped?" is: *today, only by scraping logs for `KAFKA ERROR` or noticing a metrics gap.* That's an observability gap and a legitimate senior project ([`../06-contribution-practice/03-senior-build-projects.md`](../06-contribution-practice/03-senior-build-projects.md)).

## Drills

1. **Idempotency trace:** A tracker retries the same event 3× (network flakiness). How many session rows result? How many event rows? Cite `createSession.ts:39` and `saveEvent.ts:105`.
2. **Reliability upgrade:** Design the minimal outbox change to make Kafka delivery at-least-once. What table, what relay, and what makes duplicates safe?
3. **Atomicity judgment:** For which of {pageview count, unique visitors, total revenue} would the missing event+revenue transaction actually matter, and why?

## Interview angle

- "At-least-once vs at-most-once vs exactly-once — which do you have and how?" → the Kafka gap + idempotent sessions.
- "What is idempotency and how do you achieve it in writes?" → deterministic ids + `on conflict`.
- "How would you make this reliable?" → outbox, with the duplicate tradeoff.
- "How would you know if this broke in production?" → the observability gap.
See [`../08-interview-prep/04-system-design-from-this-repo.md`](../08-interview-prep/04-system-design-from-this-repo.md) and [`../05-quality-engineering/06-observability-and-operations.md`](../05-quality-engineering/06-observability-and-operations.md).
