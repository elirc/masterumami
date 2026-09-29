# Interview Prep: JS / TS / Node Deep Dive

Fourteen question cards on JavaScript, TypeScript, and Node — the technical-screen and deep-dive rounds for a mid-level fullstack role. Most are anchored to real Umami code so you answer with evidence, not definitions. The golden rule for every answer: **concrete example + tradeoff + failure mode**, not a textbook definition.

Card format: what it tests → repo anchor → junior/mid/senior answer → follow-ups → drill.

---

## Q1: The tracker sends events with `fetch(..., { keepalive: true })`. What does `keepalive` do and why is it here?

**Round:** JS deep-dive. **What it's really testing:** understanding of the browser event loop, page lifecycle, and unload behavior.

**Repo anchor:** [`src/tracker/index.js:175-184`](../../src/tracker/index.js).

**Junior answer:** "It keeps the connection alive." (Vague.)

**Mid adds:** `keepalive` lets a `fetch` outlive the page that started it — so an event fired during `pagehide`/unload still completes instead of being cancelled when navigation tears down the document. Without it, the last pageview before a navigation is often lost.

**Senior adds:** It's the modern replacement for `navigator.sendBeacon`; it has a browser-imposed body-size cap (~64KB) shared across in-flight keepalive requests, which is *why* replay batches are capped (200 events, [`record/route.ts:23`](../../src/app/api/record/route.ts)) and events are small. Tradeoff: reliability on unload vs. size limits and no access to the response in some unload timings.

**Follow-ups:** "sendBeacon vs keepalive fetch?" "What happens if the payload exceeds the cap?"

**Drill:** Explain aloud in 60s why an analytics tracker specifically needs unload-surviving requests. Reference `flushPerformance` on `pagehide` ([`tracker/index.js:382`](../../src/tracker/index.js)).

---

## Q2: `/api/send` does several `await`s in sequence (session, event, event-data, revenue). Walk me through what the event loop is doing during that request.

**Round:** JS/Node deep-dive. **Tests:** the event loop, async/await, concurrency vs. parallelism.

**Repo anchor:** [`send/route.ts:150-271`](../../src/app/api/send/route.ts); [`saveEvent.ts:143-167`](../../src/queries/sql/events/saveEvent.ts).

**Junior:** "It waits for each line to finish." True but shallow.

**Mid adds:** Each `await` yields the thread back to the event loop; Node isn't blocked — it can serve *other* requests while this one waits on I/O (the DB round-trip). Within *this* request the awaits are serial because each depends on the previous (event id → event data → revenue).

**Senior adds:** The serialization here is partly necessary (revenue needs the event id) and partly not — independent writes could be `Promise.all`'d to cut latency. But the bigger point: there's **no transaction** wrapping event + event_data + revenue on the ClickHouse path, so a crash between them leaves partial data. For analytics that's tolerable; for money it wouldn't be. Naming that boundary is the senior move.

**Follow-ups:** "Which of these awaits could run in parallel?" "What consistency guarantee does this give you?"

**Drill:** Identify one pair of awaits in `saveEvent`'s relational branch that must stay serial and one that need not.

---

## Q3: `uuid(sourceId, ip, userAgent, salt)` returns the same id for the same inputs. How, and why does that matter?

**Round:** JS/Node + crypto basics. **Tests:** hashing vs. random ids, determinism, understanding library APIs.

**Repo anchor:** [`crypto.ts:60-66`](../../src/lib/crypto.ts) — uses `uuid` v5 (namespaced hash) when args are present, v4/v7 (random) otherwise.

**Junior:** "It's a UUID." Misses that it's deterministic.

**Mid adds:** v5 UUIDs are a hash of a name within a namespace — same inputs → same output. Umami uses this to derive a stable visitor id from IP+UA+salt without storing anything, which is the whole cookieless-identity trick.

**Senior adds:** The determinism is what makes the session insert idempotent (`on conflict do nothing`, [`createSession.ts:39`](../../src/queries/sql/sessions/createSession.ts)) and makes retries safe. The salt rotation ([`getSalt`, `crypto.ts:72`](../../src/lib/crypto.ts)) deliberately breaks determinism across periods for privacy — a designed limitation, not a bug. Tradeoff: no cross-period tracking, so returning visitors look new each month.

**Follow-ups:** "When would you use v4 vs v5 vs v7?" (v7 is time-ordered — better index locality; note `USE_UUIDV7`, `crypto.ts:65`.) "Is a hash of IP+UA PII?"

**Drill:** Explain how changing the salt every month affects the "unique visitors" metric.

---

## Q4: `parseRequest` strips unknown query keys with Zod, then re-adds keys like `browser1`, `os2`. What's going on?

**Round:** TS/validation. **Tests:** understanding schema validation defaults and reading around a workaround.

**Repo anchor:** [`request.ts:23-42`](../../src/lib/request.ts), the `/\d+$/` re-add loop (`:33-38`).

**Junior:** "It validates the query." Doesn't see the hack.

**Mid adds:** Zod's `z.object` **strips** keys not in the schema by default. Umami supports repeated filters (`browser1`, `browser2`) that aren't individually declared, so after validation it copies numeric-suffixed keys back from the raw query. It's an escape hatch around Zod stripping.

**Senior adds:** This is a smell: validated output is being *un-stripped* with unvalidated raw values, so those suffixed params bypass the schema's guarantees. Safer would be a `z.record` with a key pattern or `.catchall`. It works, but it undermines "the schema is the contract." Flag it in review.

**Follow-ups:** "How does Zod handle unknown keys — strip, strict, or passthrough?" "How would you validate an arbitrary number of repeated filters properly?"

**Drill:** Write a Zod schema that accepts `browser`, `browser1`, `browser2`, ... as strings without the re-add hack.

---

## Q5: `checkAuth` can return a user from a stateless token *or* a Redis-backed key. Explain the TypeScript and the design.

**Round:** TS + Node/sessions. **Tests:** union handling, sessions vs JWT.

**Repo anchor:** [`auth.ts:17-60`](../../src/lib/auth.ts); issuance in [`login/route.ts:36-40`](../../src/app/api/auth/login/route.ts).

**Junior:** "It checks the token." Misses the two paths.

**Mid adds:** Depending on whether `REDIS_URL` is set, the token either *contains* the `userId` (stateless, encrypted JWT) or contains an `authKey` that looks up the session in Redis (stateful). `checkAuth` handles both by branching on which field is present.

**Senior adds:** The tradeoff is revocation vs. statelessness. Stateless tokens can't be invalidated before expiry (logout only clears localStorage); the Redis path supports revocation and TTL at the cost of a lookup per request. Umami supports both by config — a pragmatic "choose per deployment" answer to the eternal sessions-vs-JWT debate.

**Follow-ups:** "How would you revoke a stateless JWT?" (deny-list, short expiry + refresh). "Why encrypt the token, not just sign it?" (Q maps to [Pattern 16](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-16-encrypted-not-just-signed-token).)

**Drill:** Given `parseSecureToken` ([`jwt.ts:20-26`](../../src/lib/jwt.ts)), explain the decrypt-then-verify order and why both steps exist.

---

## Q6: Why is `$queryRawUnsafe` not a SQL-injection vulnerability the way it's used here?

**Round:** API/security + Node. **Tests:** parameterization vs. concatenation.

**Repo anchor:** [`prisma.ts:255-282`](../../src/lib/prisma.ts) — `{{name::type}}` → `$n`, values pushed to a `params` array (`:268-276`).

**Junior:** "It looks unsafe." Can't distinguish text from values.

**Mid adds:** The `{{param}}` placeholders are rewritten to positional bind parameters (`$1`, `$2`) and the values are passed *separately* to the driver — so user values never become SQL syntax. "Unsafe" refers to the query *text* being a dynamic string, not the values.

**Senior adds:** The real risk is anything interpolated *outside* the placeholder system. `pagedRawQuery` does `order by ${orderBy}` ([`prisma.ts:318-323`](../../src/lib/prisma.ts)) — direct interpolation. It's safe *today* only because no route feeds user-controlled `orderBy` into it (the list routes hardcode `order by created_at desc` and omit `sortingParams`). That's a latent injection surface: add one route that passes user `orderBy` and you have a hole. This is the difference between "no bug" and "no vulnerability class."

**Follow-ups:** "How would you safely allow user-chosen sort columns?" (allow-list mapping, never interpolate). "Where else is text vs. value confused?"

**Drill:** Write the allow-list guard you'd add before `order by ${orderBy}`.

---

## Q7: `BigInt.prototype.toJSON` is patched in `db.ts`. Why, and what does it tell you about the JSON boundary?

**Round:** JS deep-dive. **Tests:** knowledge of `JSON.stringify` limits and number types.

**Repo anchor:** [`db.ts:7-10`](../../src/lib/db.ts).

**Junior:** "It converts BigInt." Doesn't know why it's needed.

**Mid adds:** `JSON.stringify` throws on `BigInt` ("Do not know how to serialize a BigInt"). Aggregate SQL (`count`, `sum`) returns 64-bit integers that the driver surfaces as `BigInt`; without this patch, serializing an API response would crash. The patch coerces them to `Number`.

**Senior adds:** Coercing `BigInt`→`Number` silently loses precision above `Number.MAX_SAFE_INTEGER` (2^53). For visitor counts that's fine; for something like summed revenue in minor units it could corrupt. Monkey-patching a built-in prototype globally is also a broad blast radius — it affects every serialization in the process. A scoped serializer would be safer.

**Follow-ups:** "When would `Number` coercion actually bite you?" "Alternatives to patching a prototype?"

**Drill:** Name one metric in this repo where `BigInt`→`Number` could theoretically overflow, and one where it can't.

---

## Q8: The tracker hooks `history.pushState`/`replaceState`. Explain the closure and why it's needed.

**Round:** JS deep-dive (closures, monkey-patching). **Tests:** closures, higher-order functions, SPA routing.

**Repo anchor:** [`tracker/index.js:98-109`](../../src/tracker/index.js) — `hook(this, method, callback)` returns a wrapper closing over `orig`.

**Junior:** "It overrides pushState." Can't explain the closure.

**Mid adds:** SPAs navigate via `history.pushState` without a full page load, so there's no `navigation` event to hook. The tracker replaces `pushState` with a function that closes over the original (`orig`), calls the tracking callback, then delegates to `orig` — so routing still works and each virtual navigation is tracked.

**Senior adds:** The closure preserves `orig` per-method so both `pushState` and `replaceState` are wrapped independently without losing the native implementation. Risk: if another library also wraps these, ordering matters; and this only catches programmatic navigation, so `popstate` (back button) needs separate handling. The `handlePush` guard (`:93`) avoids double-tracking when the URL didn't change.

**Follow-ups:** "How do you track back/forward navigation?" "What if React Router also patches history?"

**Drill:** Rewrite `hook` without a closure (e.g. storing originals on an object) and argue which is clearer.

---

## Q9: Explain the `.refine` in the send schema that requires "exactly one of website/link/pixel."

**Round:** TS/validation. **Tests:** Zod refinements, cross-field validation.

**Repo anchor:** [`send/route.ts:53-63`](../../src/app/api/send/route.ts).

**Junior:** "It checks the fields exist." Misses "exactly one."

**Mid adds:** Field-level rules can't express "exactly one of these three." A `.refine` runs a predicate over the whole object: it counts how many of `website/link/pixel` are truthy and requires the count to equal 1, attaching the error to a path. It encodes a mutual-exclusivity invariant.

**Senior adds:** This invariant is load-bearing downstream — `sourceId = websiteId || pixelId || linkId` ([`send/route.ts:98`](../../src/app/api/send/route.ts)) and the event-type derivation ([`:217-223`](../../src/app/api/send/route.ts)) both assume exactly one. Without the refine, a payload with two sources would silently pick one by `||` precedence — a subtle bug the schema prevents at the boundary. Validate invariants where data enters, not where it's used.

**Follow-ups:** "Where else would this invariant break if the refine were removed?" "superRefine vs refine?"

**Drill:** Find another `superRefine`/`refine` in `schema.ts` (hint: `withDateRange`, `:32-42`) and state the invariant it protects.

---

## Q10: What's a "stale closure" in React and where could Umami hit one?

**Round:** frontend/JS. **Tests:** closures over hook state, effect dependencies.

**Repo anchor:** conceptual — but React Query hooks like [`useWebsiteMetricsQuery.ts:32-38`](../../src/components/hooks/queries/useWebsiteMetricsQuery.ts) capture `startAt`, `filters` in the `queryFn`.

**Junior:** "When state is old." Vague.

**Mid adds:** A closure captures variables by reference at definition time; inside a `useCallback`/`useEffect` with a wrong dependency array, it can capture a *stale* value and keep using it after re-render. React Query mitigates this by requiring inputs to be in the `queryKey` — if `startAt` changes, the key changes and the query refetches, rather than a captured old value being reused.

**Senior adds:** The systemic defense here is that the `queryKey` *is* the dependency list — forgetting to add an input to the key is the React Query flavor of a stale closure (you serve cached data for new inputs). So "stale closure" and "incomplete query key" are the same bug in two idioms. Point at the key array as the contract.

**Follow-ups:** "How does the dependency array relate to closures?" "What breaks if `filters` isn't in the queryKey?"

**Drill:** Describe the exact wrong output if `params.type` were omitted from the metrics `queryKey`.

---

## Q11: `getClientInfo` uses `payload?.browser ?? browserName(userAgent)`. Explain `??` vs `||` and why it matters here.

**Round:** JS/TS. **Tests:** nullish coalescing, falsy pitfalls.

**Repo anchor:** [`detect.ts:133-135`](../../src/lib/detect.ts).

**Junior:** "They're the same." Wrong.

**Mid adds:** `??` falls back only on `null`/`undefined`; `||` falls back on any falsy value (`''`, `0`, `false`). Using `??` means an explicitly provided empty-string browser wouldn't be silently overridden — though here the practical effect is subtle. The habit matters more elsewhere: `pageSize || DEFAULT` ([`prisma.ts:286`](../../src/lib/prisma.ts)) treats `0` as "unset," which `??` would not.

**Senior adds:** The pattern to teach: use `||` when you truly want "any falsy → default" (e.g. `pageSize` where 0 is meaningless) and `??` when `0`/`''`/`false` are legitimate values. Mixing them up is a top source of "why is my zero being replaced?" bugs. Umami uses both deliberately in adjacent files — a good compare-and-contrast.

**Follow-ups:** "When is `||` correct and `??` wrong?" (default for a numeric that shouldn't be 0.)

**Drill:** Find a spot in `prisma.ts` where `||` is intentionally chosen so `0` becomes a default, and one where `??` would be wrong.

---

## Q12: How does TypeScript's discriminated union make the reports code safer?

**Round:** TS deep-dive. **Tests:** discriminated unions, exhaustiveness, narrowing.

**Repo anchor:** [`schema.ts:279-289`](../../src/lib/schema.ts) (`z.discriminatedUnion('type', [...])`).

**Junior:** "It's a union of types." Misses the discriminant.

**Mid adds:** A shared literal field (`type`) lets both Zod and TS narrow to the exact variant — so once you check `type === 'funnel'`, the `parameters` are typed as funnel params. Validation errors are precise, and consumers get variant-specific types.

**Senior adds:** The safety is only as good as its enforcement point. The *persisted* `parameters` column is `Json` and the base save schema uses `anyObjectParam` ([`schema.ts:271-277`](../../src/lib/schema.ts)), so the strict union guards the query/result path but not necessarily every write. The lesson: a discriminated union is a great contract, but verify it's applied at the boundary where bad data would enter, not only where it's read.

**Follow-ups:** "How would you get compile-time exhaustiveness over report types?" (`never` in a default case.) "Where is the union actually enforced vs. bypassed?"

**Drill:** Add a hypothetical new report type and list every file the discriminated union forces you to update.

---

## Q13: The Redis client is created once and stashed on `globalThis`. What Node/Next problem does that solve?

**Round:** Node/runtime. **Tests:** module caching, dev hot-reload, connection pools.

**Repo anchor:** [`redis.ts:144-151`](../../src/lib/redis.ts); same idiom in `prisma.ts:390`, `clickhouse.ts:41`.

**Junior:** "It reuses the client." Doesn't know why.

**Mid adds:** Next.js dev hot-reload re-evaluates modules on change. A module-scoped `new Client()` would create a fresh client every reload, leaking connections until the DB refuses new ones. Stashing on `globalThis` (which survives module re-evaluation) reuses one client.

**Senior adds:** It's guarded to non-production (`NODE_ENV !== 'production'`) because prod doesn't hot-reload and you don't want global leakage there. Downside: a cached global can mask config changes until a full restart, and typing `globalThis[X]` is untyped. The connection-management concept generalizes to any long-lived client (DB, message broker) in a serverless/hot-reloading runtime.

**Follow-ups:** "Why gate it to non-production?" "What symptom tells you this pattern is missing?" (pool exhaustion after many reloads).

**Drill:** Explain the exact error a developer sees after ~50 hot reloads without this pattern.

---

## Q14: `serializeError` is used in the send/record catch blocks instead of returning the raw error. Why?

**Round:** Node/error handling + security. **Tests:** error serialization, information leakage.

**Repo anchor:** [`send/route.ts:314-321`](../../src/app/api/send/route.ts), [`record/route.ts:117-124`](../../src/app/api/record/route.ts).

**Junior:** "To log the error." Partial.

**Mid adds:** `Error` objects don't serialize cleanly to JSON (non-enumerable `message`/`stack`), so `serializeError` turns them into a plain, loggable/returnable object. It's used both to `console.log` and to include an `errorObject` in the response.

**Senior adds:** Returning a serialized error to the *client* (`serverError({ errorObject })`) risks leaking internals (stack traces, query fragments) to an unauthenticated caller on a public endpoint. That's an information-disclosure smell worth flagging: log the full error server-side, return a generic message client-side. The double-use here (log + return) conflates those audiences.

**Follow-ups:** "What shouldn't a 500 response body contain?" "How do you correlate a client 500 with a server log without leaking?" (an error id).

**Drill:** Rewrite the catch to log the full serialized error but return only `{ error: 'internal', id }` to the client.

---

## How to drill this module

Do **three cards a day** aloud, timed to 90 seconds each, always ending with a failure mode. Then open the anchor and confirm you cited it correctly. Cards Q2, Q6, Q9, and Q12 are the ones most likely to escalate into follow-up chains — over-prepare those. Cross-reference the framework questions in [`02-frontend-framework-questions.md`](02-frontend-framework-questions.md) and the API questions in [`03-api-and-data-modeling-questions.md`](03-api-and-data-modeling-questions.md).
