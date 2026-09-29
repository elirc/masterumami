# Fake-Code Contrasts

Ten bad-vs-better pairs. The "bad" version shows a novice instinct; the "better" version shows the shape Umami actually uses (or should). **All code below is illustrative and labelled — it is not copied from the repo.** Each pair ends with the real anchor to compare against.

The goal: train your eye to *feel* the bad shape before you can articulate why. In review, that instinct is what flags a problem before you've finished reading.

---

## 1. Coupling UI shape to DB shape

```ts
// Illustrative fake code: not from this repo
// BAD: component renders raw DB rows
function Sessions({ rows }) {
  return rows.map(r => <div>{r.session_id} {r.website_id} {r.created_at}</div>);
}
```

```ts
// Illustrative fake code: not from this repo
// BETTER: API returns a shaped, aliased result; UI consumes a typed view model
// (real queries alias snake_case → camelCase at the SQL boundary)
function Sessions({ sessions }: { sessions: SessionView[] }) {
  return sessions.map(s => <SessionRow key={s.id} session={s} />);
}
```

**Why better:** the DB's `snake_case` and column names shouldn't leak to the client. Umami aliases at the SQL boundary (`session.session_id as "id"`, [`getWebsiteSessions.ts:36-52`](../../src/queries/sql/sessions/getWebsiteSessions.ts)) so the API contract is independent of the schema. Compare: real code.

---

## 2. Missing permission filter (IDOR)

```ts
// Illustrative fake code: not from this repo
// BAD: authenticate but don't authorize the resource
const { auth } = await parseRequest(request);
const data = await getWebsiteStats(websiteId, filters); // any authed user, any website!
return json(data);
```

```ts
// Illustrative fake code: not from this repo
// BETTER: authorize the specific resource before querying
const { auth, error } = await parseRequest(request, schema);
if (error) return error();
if (!(await canViewWebsite(auth, websiteId))) return unauthorized();
const data = await getWebsiteStats(websiteId, filters);
```

**Why better:** authentication ≠ authorization. The real routes always gate with `can*` ([`realtime/[websiteId]/route.ts:20`](../../src/app/api/realtime/[websiteId]/route.ts)). Omitting it is the #1 risk class here ([`../03-architecture-and-patterns/03-validation-auth-and-permissions.md`](../03-architecture-and-patterns/03-validation-auth-and-permissions.md)).

---

## 3. N+1 query

```ts
// Illustrative fake code: not from this repo
// BAD: one query per website
const websites = await getWebsites();
for (const w of websites) {
  w.stats = await getWebsiteStats(w.id, filters); // N round-trips
}
```

```ts
// Illustrative fake code: not from this repo
// BETTER: one aggregate query grouped by website, or a bounded page
const stats = await getStatsForWebsites(websiteIds, filters); // 1 round-trip
```

**Why better:** N+1 is the classic scaling killer. Umami's aggregations do the grouping in SQL in a single query (`getWebsiteStats.ts` computes five metrics in one grouped scan). If you ever see a loop with an `await` calling the DB, suspect N+1.

---

## 4. Stale cache key (React Query)

```ts
// Illustrative fake code: not from this repo
// BAD: queryKey omits an input the result depends on
useQuery({ queryKey: ['metrics', websiteId], queryFn: () => get(url, { startAt, endAt, type }) });
// changing `type` shows stale data for the old type
```

```ts
// Illustrative fake code: not from this repo
// BETTER: every input in the key
useQuery({ queryKey: ['metrics', { websiteId, startAt, endAt, type }], queryFn: ... });
```

**Why better:** the key *is* the cache identity. Real hooks include every input ([`useWebsiteMetricsQuery.ts:22-31`](../../src/components/hooks/queries/useWebsiteMetricsQuery.ts)). Omitting one is the React Query flavor of a stale closure.

---

## 5. Side effect in an unreliable place, swallowed

```ts
// Illustrative fake code: not from this repo
// BAD: fire a critical write and ignore failure, return success
try { await queue.publish(event); } catch {}
return json({ ok: true }); // lied: the event may be gone
```

```ts
// Illustrative fake code: not from this repo
// BETTER: durable outbox, relay retries, response reflects acceptance
await db.outbox.insert(event);   // durable
return json({ ok: true });        // truthful: we durably accepted it
```

**Why better:** this is exactly Umami's Kafka gap ([`kafka.ts:87-90`](../../src/lib/kafka.ts)) — the "bad" shape is real here. The fix is an outbox ([`../03-architecture-and-patterns/04-side-effects-async-and-reliability.md`](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md)). Recognize "empty catch after a critical write returning success" as a reliability smell.

---

## 6. Swallowed error hiding a bug

```ts
// Illustrative fake code: not from this repo
// BAD: swallow and return a default, hiding the cause
function parse(token) { try { return verify(token); } catch { return {}; } }
// caller treats {} as "anonymous but valid" — a security footgun
```

```ts
// Illustrative fake code: not from this repo
// BETTER: return an explicit null the caller must handle
function parseToken(token, secret) { try { return jwt.verify(token, secret); } catch { return null; } }
```

**Why better:** Umami returns `null` on bad tokens ([`jwt.ts:8-14`](../../src/lib/jwt.ts)) and callers branch on it (`if (!user?.id && !shareToken) return null`, [`auth.ts:37`](../../src/lib/auth.ts)). Returning `{}` would be interpreted as a valid-but-empty auth — a dangerous fail-*open*. `null` forces a decision.

---

## 7. String-concatenated SQL (injection)

```ts
// Illustrative fake code: not from this repo
// BAD: user value in the SQL text
const rows = await db.query(`select * from website_event where url_path = '${path}'`);
```

```ts
// Illustrative fake code: not from this repo
// BETTER: named placeholder → bound parameter
const rows = await rawQuery(
  `select * from website_event where url_path = {{path}}`, { path }
);
```

**Why better:** Umami's engine binds values, never concatenates ([`prisma.ts:268-276`](../../src/lib/prisma.ts)). The one place to stay alert is non-value interpolation like `order by ${orderBy}` (`prisma.ts:318`) — allow-list those.

---

## 8. `any` spreading instead of narrowing

```ts
// Illustrative fake code: not from this repo
// BAD: any leaks through, no safety
function handle(body: any) { return save(body.payload.website); } // typos compile
```

```ts
// Illustrative fake code: not from this repo
// BETTER: validate at the edge, get a typed object
const result = schema.safeParse(body);
if (!result.success) return badRequest(...);
return save(result.data.payload.website); // typed, coerced
```

**Why better:** Umami validates at the boundary and works with typed `result.data` ([`request.ts:26-41`](../../src/lib/request.ts)). `any` should stop at the edge, not flow inward. (The raw-SQL layer's `any` is a *deliberate, contained* exception — [`../02-stack-and-language-mastery/03-type-system-and-contracts.md`](../02-stack-and-language-mastery/03-type-system-and-contracts.md).)

---

## 9. `||` where `??` is meant

```ts
// Illustrative fake code: not from this repo
// BAD: a valid 0 gets replaced
const sampleRate = config.rate || 0.15; // rate=0 (disable) becomes 0.15!
```

```ts
// Illustrative fake code: not from this repo
// BETTER: only null/undefined falls back
const sampleRate = config.rate ?? 0.15; // rate=0 stays 0
```

**Why better:** `||` treats `0`/`''`/`false` as "unset." Umami uses `||` intentionally where 0 is meaningless (`pageSize || DEFAULT`, [`prisma.ts:286`](../../src/lib/prisma.ts)) and `??` where a falsy value is valid ([`detect.ts:133`](../../src/lib/detect.ts)). Choose per meaning, not by habit.

---

## 10. Casual public-contract change

```ts
// Illustrative fake code: not from this repo
// BAD: rename a payload field "for clarity"
const schema = z.object({ payload: z.object({ websiteId: z.uuid() /* was: website */ }) });
// every deployed tracker still sends `website` → silent validation failures
```

```ts
// Illustrative fake code: not from this repo
// BETTER: keep the public field; accept both during a deprecation window
const schema = z.object({ payload: z.object({
  website: z.uuid().optional(), websiteId: z.uuid().optional(),
}).refine(p => p.website ?? p.websiteId) });
```

**Why better:** the `/api/send` payload is a public contract depended on by deployed scripts ([`send/route.ts:24-64`](../../src/app/api/send/route.ts)). Renaming a field breaks every embedder silently. Additive, backward-compatible evolution is the only safe path for public contracts.

---

## Debrief

The meta-skill: each "bad" shape has a *feel* — a loop with an await (N+1), an empty catch before a success return (silent loss), a value inside a template string in SQL (injection), a `queryKey` missing an input (stale). Train the feel, and you'll flag these in review before you've reasoned them out. Several of these ("bad" #2, #5, #9) are real live tradeoffs in Umami, not hypotheticals — which is what makes them worth internalizing. Continue to [`04-review-katas.md`](04-review-katas.md).
