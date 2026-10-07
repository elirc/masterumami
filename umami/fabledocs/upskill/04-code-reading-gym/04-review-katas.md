# Review Katas

Ten fake PRs to review. For each: read the intent and diff summary, then write your findings graded **Blocking / Important / Optional** before reading the expected findings. Practice kind, specific review language — the example comments model it.

All diffs are illustrative and described, not real commits. The files they *resemble* are anchored so you can compare against real code.

Grading key: **Blocking** = must fix before merge (correctness/security). **Important** = should fix (maintainability/risk). **Optional** = nit/preference.

---

## Kata 1: "Add sorting to the sessions list"

**Author intent:** let users sort the sessions table by column.

**Diff summary:** adds `orderBy` to the sessions route schema (`sortingParams`) and passes `query.orderBy` through `getQueryFilters` into `getWebsiteSessions` → `pagedRawQuery`.

**Resembles:** [`sessions/route.ts`](../../../src/app/api/websites/[websiteId]/sessions/route.ts), [`prisma.ts:318-323`](../../../src/lib/prisma.ts).

**Your task:** review it.

**Expected findings —**
- **Blocking:** `pagedRawQuery` interpolates `order by ${orderBy}` directly into SQL. Wiring user-controlled `orderBy` in creates a SQL-injection vulnerability. Require an allow-list of sortable columns.
- **Important:** the sessions query already hardcodes `order by max(created_at) desc`; adding an `orderBy` may conflict or be ignored — clarify the interaction.
- **Optional:** add a test for the allow-list rejection path.

**Good comment:**
> Blocking: this passes user input into `order by ${orderBy}` in `pagedRawQuery` (prisma.ts:318), which is string-interpolated, not bound — that's an injection vector. Could we map the requested column through an allow-list (`{ createdAt: 'created_at', ... }`) and reject anything else? Happy to pair on it.

---

## Kata 2: "Speed up collection by parallelizing writes"

**Author intent:** reduce latency in `saveEvent` by running writes concurrently.

**Diff summary:** wraps the event, event-data, and revenue writes in `Promise.all`.

**Resembles:** [`saveEvent.ts:143-167`](../../../src/queries/sql/events/saveEvent.ts).

**Expected findings —**
- **Blocking:** revenue and event_data both need the event's `websiteEventId`, which is created by the event insert. `Promise.all`-ing them with the event insert races the id — they'd run before the event exists. The event write must complete first.
- **Important:** event_data and revenue *could* run concurrently *with each other* after the event insert — that's the safe optimization.
- **Optional:** measure before claiming a speedup; these are already fast inserts.

**Good comment:**
> The event insert has to finish first — `saveEventData`/`saveRevenue` use `websiteEventId` from it (saveEvent.ts:147,157). What if we `await` the event, then `Promise.all([saveEventData(...), saveRevenue(...)])`? That gets the concurrency safely.

---

## Kata 3: "Cache the website lookup in the record endpoint"

**Author intent:** the record endpoint queries `getWebsite` on every request; cache it like collection does.

**Diff summary:** replaces `getWebsite(websiteId)` with `fetchWebsite(websiteId)` in `record/route.ts`.

**Resembles:** [`record/route.ts:56-57`](../../../src/app/api/record/route.ts), [`load.ts:6`](../../../src/lib/load.ts).

**Expected findings —**
- **Blocking:** the direct query is *intentional* — there's a comment "avoid stale Redis cache for recordingEnabled." Caching reintroduces up-to-24h staleness on `replayEnabled`, so disabling replay wouldn't take effect. This regresses a deliberate correctness choice.
- **Optional:** if caching is truly desired, it must be paired with cache invalidation on website update.

**Good comment:**
> This bypass is deliberate (there's a comment at line 56): `replayEnabled` must be read fresh so toggling replay off takes effect immediately. `fetchWebsite` caches for 24h, so this would delay disabling replay by up to a day. I'd keep the direct read unless we also invalidate the cache on update.

---

## Kata 4: "Return the DB error to help debugging"

**Author intent:** make 500s more helpful by including the error message in the response.

**Diff summary:** changes a route's catch to `return serverError({ message: e.message, stack: e.stack })`.

**Resembles:** error handling in [`send/route.ts:314-320`](../../../src/app/api/send/route.ts).

**Expected findings —**
- **Blocking (security):** returning `stack` to the client leaks internals (file paths, query fragments) — information disclosure, worse on an unauthenticated endpoint. Log server-side, return a generic message + an error id.
- **Important:** the existing pattern already `console.log`s a serialized error; extend that, don't expose it.

**Good comment:**
> Let's not send `stack` to the client — it exposes internal paths/queries, and this endpoint is public. Could we log the full error server-side (we already `serializeError` + console.log) and return `{ error: 'internal', id }` so support can correlate?

---

## Kata 5: "Add a rate limit to login"

**Author intent:** prevent brute-forcing passwords.

**Diff summary:** calls `redis.rateLimit(\`login:${username}\`, 5, 60)` at the top of the login handler; returns 429 when exceeded.

**Resembles:** [`login/route.ts`](../../../src/app/api/auth/login/route.ts), [`redis.ts:72-82`](../../../src/lib/redis.ts).

**Expected findings —**
- **Important:** good idea, but keying by `username` lets an attacker lock out a victim (denial of service) and doesn't limit password spraying across many usernames from one IP. Consider keying by IP (or IP+username).
- **Important:** `rateLimit` requires Redis; guard for `redis.enabled` or the call throws when Redis is off.
- **Optional:** 429 message shouldn't reveal whether the username exists.

**Good comment:**
> Nice — one thing: keying on username alone means I can lock out any user by spamming their name. What about `ip` (or `ip:username`)? Also this needs a `redis.enabled` guard so non-Redis deployments don't break.

---

## Kata 6: "Simplify: remove the exactly-one-source refine"

**Author intent:** the refine seems redundant since `sourceId = website || pixel || link` handles it.

**Diff summary:** deletes the `.refine` from the send schema.

**Resembles:** [`send/route.ts:53-63`](../../../src/app/api/send/route.ts).

**Expected findings —**
- **Blocking:** removing the refine lets a payload with two sources through; `sourceId` silently picks one by `||` precedence, and the event-type derivation (`:217-223`) misclassifies the event. The refine encodes a real invariant multiple downstream lines depend on.
- **Optional:** add a test asserting a two-source payload is rejected, to prevent this "cleanup" recurring.

**Good comment:**
> This refine isn't redundant — without it, `{ website, pixel }` both set would pass, and `sourceId`/event-type would silently pick one (send/route.ts:98,217). It's guarding an invariant. I'd keep it and add a test so its purpose is obvious.

---

## Kata 7: "Add a helper and use it everywhere"

**Author intent:** DRY up the `https://${hostname}` base-URL construction.

**Diff summary:** extracts `getBase(hostname)` and replaces three inline usages. No behavior change.

**Resembles:** [`send/route.ts:178,283`](../../../src/app/api/send/route.ts).

**Expected findings —**
- **Optional (approve):** legitimate small DRY, follows the codebase style, no behavior change. This is a *good* PR.
- **Important:** ensure the extracted helper preserves the exact fallback (`hostname ? ... : 'https://localhost'`) — verify by diffing behavior, and confirm no test regressions.

**Good comment:**
> LGTM — clean extraction. Just double-checking the fallback is byte-for-byte the same (`'https://localhost'` when hostname is absent). If tests are green, ship it.

---

## Kata 8: "Store custom event data as JSON instead of EAV"

**Author intent:** the `event_data` flattening is complex; store the raw JSON blob in one column instead.

**Diff summary:** replaces `saveEventData`'s row-per-key insert with a single `jsonb` column write.

**Resembles:** [`saveEventData.ts:27-48`](../../../src/queries/sql/events/saveEventData.ts).

**Expected findings —**
- **Blocking:** this breaks every query that filters/aggregates on `data_key`/`string_value` (funnels with property filters, event-data reports). The EAV shape exists precisely to make properties *queryable*; a blob isn't.
- **Blocking:** schema migration + backfill required for existing data; not addressed.
- **Important:** this is an architecture change, not a refactor — needs an RFC, not a PR.

**Good comment:**
> This changes a load-bearing design decision. `event_data` is EAV so we can filter/aggregate on properties (see getFunnel's property filters, getEventData*). A JSONB blob loses that. This deserves an RFC covering query impact, migration, and backfill before code — want to write one together?

---

## Kata 9: "Fix typo: referer → referrer"

**Author intent:** correct the misspelled header read in the pixel route.

**Diff summary:** changes `request.headers.get('referer')` to `'referrer'`.

**Resembles:** [`p/[slug]/route.ts:50`](../../../src/app/(collect)/p/[slug]/route.ts).

**Expected findings —**
- **Blocking:** `Referer` is the *correct*, spec-defined HTTP header name (the misspelling is in the HTTP standard itself). Changing it to `referrer` means the header is never found → referrer tracking silently breaks.
- **Optional:** add a comment noting the intentional spelling so this doesn't recur.

**Good comment:**
> Careful — `Referer` (one 'r') is the actual HTTP header name; it's a famous spec typo. `headers.get('referrer')` would return null and break referrer capture. Let's revert and add a `// HTTP header is intentionally misspelled` comment.

---

## Kata 10: "Add index to speed up a slow query"

**Author intent:** the sessions query is slow; add an index on `website_event.created_at`.

**Diff summary:** adds `@@index([createdAt])` to `WebsiteEvent` (which already has many composite indexes).

**Resembles:** [`schema.prisma:135-149`](../../../prisma/schema.prisma).

**Expected findings —**
- **Important:** `@@index([createdAt])` already exists (`:135`), and queries filter by `website_id` *first*, so a lone `created_at` index won't help — the composite `(website_id, created_at, ...)` indexes serve those. Measure with `EXPLAIN` before adding.
- **Important:** every added index slows writes (this is a write-heavy table). Adding a redundant one is a net negative.
- **Optional:** the real fix is likely bounding the date range / page size, not indexing.

**Good comment:**
> I don't think this helps — we already have `@@index([createdAt])` (schema.prisma:135), and the sessions query filters by `website_id` first, so it uses the composite indexes. Adding indexes also slows inserts on this hot table. Could we `EXPLAIN ANALYZE` the actual query first? I suspect the cost is an unbounded date range, not a missing index.

---

## Debrief

The review skill isn't finding *something* to say — it's grading severity correctly and saying it kindly. Notice the katas where the "fix" is actually a regression (3, 6, 9) or an architecture change in disguise (8): a good reviewer protects the codebase from plausible-looking mistakes. And notice Kata 7, where the right call is **approve** — a reviewer who blocks everything is as useless as one who blocks nothing. Take this into [`../07-career-and-collaboration/01-code-review-mindset.md`](../07-career-and-collaboration/01-code-review-mindset.md) and the interview review round ([`../08-interview-prep/05-debugging-and-code-review-rounds.md`](../08-interview-prep/05-debugging-and-code-review-rounds.md)).
