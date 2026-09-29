# Annotation Drills

Ten real excerpts from Umami. For each: annotate **inputs, outputs, dependencies, invariants, side effects, and failure modes** before reading the debrief. The debrief is a self-grading rubric, not just an answer key. Do the work first — reading the answer without annotating teaches nothing.

How to annotate: open the anchored file, read the excerpt in context, and write one line for each of the six dimensions. Then grade yourself Basic/Solid/Strong against the debrief.

---

## Drill 1 — The identity derivation

**Anchor:** [`src/app/api/send/route.ts:143-175`](../../src/app/api/send/route.ts)

```ts
const saltRotation = process.env.SALT_ROTATION || 'month';
const sessionSalt = getSalt(saltRotation, createdAt);
const visitSalt = hash(startOfHour(createdAt).toUTCString());
const sessionId = id ? uuid(sourceId, id) : uuid(sourceId, ip, userAgent, sessionSalt);
// ...
let visitId = cache?.visitId || uuid(sessionId, visitSalt);
let iat = cache?.iat || now;
if (!timestamp && now - iat > 1800) { visitId = uuid(sessionId, visitSalt); iat = now; }
```

**Annotate, then check:**
- **Inputs:** `sourceId`, `ip`, `userAgent`, `createdAt`, optional explicit `id`, optional `cache`.
- **Outputs:** `sessionId`, `visitId`, `iat`.
- **Dependencies:** `getSalt`, `uuid` (v5 hash), `hash`, `startOfHour`, env `SALT_ROTATION`.
- **Invariant:** same visitor + same salt window → same `sessionId`; a visit expires after 1800s of inactivity.
- **Side effects:** none here (pure derivation).
- **Failure modes:** shared IP/UA collapses users; salt boundary resets identity; if `timestamp` is provided (backfill), the 30-min expiry is skipped (`!timestamp` guard).

**Self-grade:** *Basic* — you named session/visit ids as outputs. *Solid* — you spotted the salt window as the invariant. *Strong* — you noticed the `!timestamp` guard means historical/backfilled events never expire a visit, and can say why that's deliberate.

---

## Drill 2 — The persistence dispatcher

**Anchor:** [`src/lib/db.ts:22-36`](../../src/lib/db.ts)

```ts
export async function runQuery(queries: any) {
  if (process.env.CLICKHOUSE_URL) {
    if (queries[KAFKA]) return queries[KAFKA]();
    return queries[CLICKHOUSE]();
  }
  const db = getDatabaseType();
  if (db === POSTGRESQL) return queries[PRISMA]();
}
```

**Check:**
- **Inputs:** a map of backend → thunk.
- **Outputs:** the result of exactly one thunk.
- **Dependencies:** env `CLICKHOUSE_URL`, `DATABASE_URL` (via `getDatabaseType`).
- **Invariant:** exactly one backend runs; Kafka is preferred over direct ClickHouse when present.
- **Side effects:** whatever the chosen thunk does (a DB write/read).
- **Failure modes:** if `CLICKHOUSE_URL` unset and `db` isn't `postgresql`, the function returns `undefined` silently — a misconfiguration fails quietly.

**Self-grade:** *Solid* if you saw "exactly one backend." *Strong* if you flagged the silent `undefined` return on misconfiguration as a failure mode.

---

## Drill 3 — The permission check (IDOR guard)

**Anchor:** [`src/permissions/website.ts:7-40`](../../src/permissions/website.ts)

```ts
export async function canViewWebsite({ user, shareToken }: Auth, websiteId: string) {
  if (user?.isAdmin) return true;
  if (shareToken?.websiteId === websiteId || /* ...more id matches... */) return true;
  const entity = await getEntity(websiteId);
  if (!entity || !user) return false;
  if (entity.userId) return user.id === entity.userId;
  if (entity.teamId) { const tu = await getTeamUser(entity.teamId, user.id); return !!tu; }
  return false;
}
```

**Check:**
- **Inputs:** `auth` (user + shareToken), `websiteId`.
- **Outputs:** boolean.
- **Dependencies:** `getEntity`, `getTeamUser`.
- **Invariant:** access requires admin OR matching share scope OR ownership OR team membership; default deny.
- **Side effects:** DB reads.
- **Failure modes:** if a new caller forgets to invoke this, IDOR; `getEntity` and `getWebsite` (used by sibling `canUpdateWebsite`) are separate reads that could diverge.

**Self-grade:** *Solid* if you named "default deny." *Strong* if you noticed the ordering (admin → share → owner → team) is a precedence decision and that the systemic risk is a *forgotten call*, not a bug in this function.

---

## Drill 4 — The event write, relational branch

**Anchor:** [`src/queries/sql/events/saveEvent.ts:143-167`](../../src/queries/sql/events/saveEvent.ts)

```ts
if (eventData) {
  await saveEventData({ websiteId, sessionId, eventId: websiteEventId, urlPath, eventName, eventData, createdAt });
  const { revenue, currency } = eventData;
  if (revenue > 0 && currency) {
    await saveRevenue({ websiteId, sessionId, eventId: websiteEventId, eventName, currency, revenue, createdAt });
  }
}
```

**Check:**
- **Inputs:** `eventData` (custom props), the just-created `websiteEventId`.
- **Outputs:** none (writes).
- **Dependencies:** `saveEventData`, `saveRevenue`.
- **Invariant:** revenue is recorded only when `revenue > 0 && currency` present.
- **Side effects:** 1–2 additional table writes after the event insert.
- **Failure modes:** **no transaction** wraps event + event_data + revenue — a crash between them leaves partial data; `revenue` is trusted from client `eventData` (spoofable).

**Self-grade:** *Solid* if you saw the two child writes. *Strong* if you flagged both the missing transaction boundary and that revenue is client-supplied.

---

## Drill 5 — Cache-aside fetch with tombstone

**Anchor:** [`src/lib/redis.ts:84-102`](../../src/lib/redis.ts)

```ts
async fetch(key, query, time?) {
  const result = await this.get(key);
  if (result === DELETED) return null;
  if (!result && query) {
    const data = await query();
    if (data) { await this.set(key, data, time); }
    return data;
  }
  return result;
}
async remove(key, soft = false) { return soft ? this.set(key, DELETED) : this.del(key); }
```

**Check:**
- **Inputs:** `key`, a `query` thunk, optional TTL.
- **Outputs:** cached value, freshly queried value, or `null` (tombstoned).
- **Dependencies:** Redis get/set.
- **Invariant:** a soft-removed key returns `null` and is not repopulated by a racing `fetch`.
- **Side effects:** may write to Redis.
- **Failure modes:** falsy-but-valid query results (`0`, `''`) won't be cached (`if (data)`); TTL staleness for values that change without a `remove`.

**Self-grade:** *Solid* if you explained the tombstone. *Strong* if you noticed `if (data)` means a legitimately falsy value is never cached, causing repeated queries.

---

## Drill 6 — The suffixed-filter re-add

**Anchor:** [`src/lib/request.ts:30-42`](../../src/lib/request.ts)

```ts
} else if (isGet) {
  query = result.data;
  for (const key of Object.keys(rawQuery)) {
    if (/\d+$/.test(key) && !(key in query)) { query[key] = rawQuery[key]; }
  }
}
```

**Check:**
- **Inputs:** validated `result.data`, original `rawQuery`.
- **Outputs:** `query` with numeric-suffixed keys re-added.
- **Dependencies:** Zod having stripped unknown keys.
- **Invariant:** repeated filters (`browser1`, `os2`) survive validation.
- **Side effects:** none.
- **Failure modes:** re-added values are **unvalidated** raw strings — they bypass the schema's guarantees; this is a workaround around Zod's default stripping.

**Self-grade:** *Solid* if you saw why the loop exists. *Strong* if you called it a smell (validated data being un-stripped with unvalidated values) and proposed `z.record`/`.catchall`.

---

## Drill 7 — Fire-and-forget Kafka send

**Anchor:** [`src/lib/kafka.ts:66-91`](../../src/lib/kafka.ts)

```ts
async function sendMessage(topic, message) {
  try {
    await connect();
    return producer.send({ topic, messages: /* ... */, timeout: SEND_TIMEOUT, acks: ACKS });
  } catch (e) {
    console.log('KAFKA ERROR:', serializeError(e));
  }
}
```

**Check:**
- **Inputs:** `topic`, one or many messages.
- **Outputs:** `RecordMetadata[]` on success, `undefined` on failure.
- **Dependencies:** `connect`, a module-level `producer`.
- **Invariant:** *none guaranteed* — delivery is best-effort.
- **Side effects:** network send to Kafka; console log on error.
- **Failure modes:** a broker outage is swallowed → **silent event loss with a success response** upstream; `acks: 1` means no full-replication guarantee.

**Self-grade:** *Solid* if you saw errors are swallowed. *Strong* if you connected it to at-most-once delivery and the upstream 200 response, and named the outbox alternative.

---

## Drill 8 — The tracker's history hook

**Anchor:** [`src/tracker/index.js:98-109`](../../src/tracker/index.js)

```js
const handlePathChanges = () => {
  const hook = (_this, method, callback) => {
    const orig = _this[method];
    return (...args) => { callback.apply(null, args); return orig.apply(_this, args); };
  };
  history.pushState = hook(history, 'pushState', handlePush);
  history.replaceState = hook(history, 'replaceState', handlePush);
};
```

**Check:**
- **Inputs:** the `history` object and method names.
- **Outputs:** wrapped methods installed on `history`.
- **Dependencies:** `handlePush`, browser `history` API.
- **Invariant:** the original method still runs after the callback (routing not broken).
- **Side effects:** mutates the global `history` methods.
- **Failure modes:** only catches programmatic navigation (not back/forward `popstate`); conflicts if another lib also wraps `history`; the closure over `orig` per method is essential.

**Self-grade:** *Solid* if you explained the closure over `orig`. *Strong* if you noted `popstate` isn't covered and the double-wrap conflict risk.

---

## Drill 9 — The exactly-one-source refine

**Anchor:** [`src/app/api/send/route.ts:53-63`](../../src/app/api/send/route.ts)

```ts
.refine(data => {
  const keys = [data.website, data.link, data.pixel];
  return keys.filter(Boolean).length === 1;
}, { message: 'Exactly one of website, link, or pixel must be provided', path: ['website'] });
```

**Check:**
- **Inputs:** the whole payload object.
- **Outputs:** validation pass/fail.
- **Dependencies:** Zod `.refine`.
- **Invariant:** exactly one event source.
- **Side effects:** none.
- **Failure modes:** if removed, `sourceId = websiteId || pixelId || linkId` ([`:98`](../../src/app/api/send/route.ts)) silently picks one by precedence, and event-type derivation ([`:217-223`](../../src/app/api/send/route.ts)) misclassifies.

**Self-grade:** *Solid* if you named the invariant. *Strong* if you traced the two downstream places that assume it and explained the silent bug its removal would cause.

---

## Drill 10 — The cache-bypass in replay ingest

**Anchor:** [`src/app/api/record/route.ts:56-65`](../../src/app/api/record/route.ts)

```ts
// Query directly to avoid stale Redis cache for recordingEnabled
const website = await getWebsite(websiteId);
if (!website) return badRequest({ message: 'Website not found.' });
if (!website.replayEnabled) return json({ ok: false, reason: 'replay_disabled' });
```

**Check:**
- **Inputs:** `websiteId`.
- **Outputs:** the website or an early return.
- **Dependencies:** `getWebsite` (not the cached `fetchWebsite`).
- **Invariant:** `replayEnabled` must be read fresh, never from cache.
- **Side effects:** a DB read on every record request (no cache benefit).
- **Failure modes:** trades latency for correctness; the collection path ([Flow 1](../01-codebase-cartography/05-key-flows.md#flow-1-collection--the-write-hot-path)) makes the *opposite* trade (cached, up to 24h stale).

**Self-grade:** *Solid* if you saw the deliberate bypass. *Strong* if you contrasted it with `fetchWebsite` in the collection path and articulated the latency-vs-correctness decision as per-flow.

---

## Debrief

If you scored *Strong* on 7+ drills you read code like a mid-level engineer: you don't just say what the code does, you name its invariants, side effects, and failure modes. The dimensions you most often missed (probably: invariants and failure modes) are exactly what interviewers probe with "what could go wrong here?" Continue to [`02-trace-tables.md`](02-trace-tables.md) to practice following data across files.
