# Risk Register

Verified observations about Umami, ranked. **Confidence** distinguishes confirmed facts from hypotheses. Nothing here is asserted as an exploited bug — items are risks, tradeoffs, or seams a senior would inspect. Each links to fuller treatment. Cross-reference [`verification-log.md`](verification-log.md) for how each was checked.

Severity = potential impact × reach. Likelihood = how easily it manifests.

| # | Risk | Severity | Likelihood | Confidence | Evidence (anchor) | Suggested test | Suggested fix |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | **Fire-and-forget Kafka drops events silently** — broker outage → event lost, 200 returned | High | Medium (on outage) | Confirmed | `kafka.ts:87-90` | Stub `producer.send` to throw; assert current swallow, then assert new guarantee | Outbox + relay (at-least-once); duplicates absorbed by deterministic ids — Project S1 |
| 2 | **Per-handler authorization invites IDOR** — a new `[id]` route can forget its `can*` check | High | Low-Med (grows with routes) | Confirmed (pattern) | `permissions/website.ts:7`, manual calls e.g. `realtime/[websiteId]/route.ts:20` | Meta-test enumerating `[id]` routes; cross-tenant 401 e2e | `withAuthorization` wrapper or lint rule — Project S2 |
| 3 | **No app-level rate limiting on open ingest** — `/send`,`/batch`,`/record` unauthenticated + unthrottled | High | Medium | Confirmed (uncalled helper) | `redis.rateLimit` exists `redis.ts:72`, no caller in `src/app/api` | Load test; unit the limiter branch | Call `rateLimit` keyed by IP+website — Ticket M1 |
| 4 | **`/batch` amplification** — each item re-enters full `send`; array only Zod-bounded, not size-capped | Med-High | Medium | Confirmed | `batch/route.ts:21-46`, `:7` | Assert over-cap array rejected | Add `.max(N)` to the array — Ticket 10 |
| 5 | **Latent SQL injection in `pagedRawQuery`** — `order by ${orderBy}` interpolated; safe only because no route feeds user input | Medium | Low (today) | Confirmed (not currently reachable) | `prisma.ts:318-323`; verified list routes hardcode order & omit `sortingParams` | Assert non-allow-listed `orderBy` rejected | Allow-list sortable columns — Ticket 6 / M3 |
| 6 | **No atomicity across event/event_data/revenue** — crash mid-write leaves partial data | Medium | Low | Confirmed | `saveEvent.ts:143-167` | Simulate failure between writes | Wrap in a transaction *if* revenue becomes authoritative; else accept |
| 7 | **Cache staleness for non-tombstoned changes** — 24h TTL serves stale `replayEnabled`/website flags on cached paths | Medium | Medium | Confirmed | `load.ts:10` (TTL) vs deliberate bypass `record/route.ts:56` | Assert update invalidates cache | Invalidate cache on website update — Ticket M4 |
| 8 | **Error responses leak internals** — collection catch returns `serializeError(e)` to unauth clients | Medium | Medium | Confirmed | `send/route.ts:314-320`, `record/route.ts:117-124` | Force 500; assert generic body | Log server-side, return `{error,id}` — Ticket M5 |
| 9 | **Validation bypass via suffixed-filter re-add** — `browser1` etc. re-added post-Zod, unvalidated | Low-Med | Medium | Confirmed | `request.ts:33-38` | Assert suffixed filter validated | `z.record`/`.catchall` pattern |
| 10 | **Client-supplied `revenue` is unauthenticated** — anyone can POST fake revenue for a known website id | Medium | High (trivially) | Confirmed | `saveEvent.ts:154`, payload `send/route.ts:24` | — (inherent to open ingest) | Don't treat client revenue as authoritative; document limit |
| 11 | **Report `parameters` may bypass strict validation** — save path uses `anyObjectParam` while a discriminated union exists | Low-Med | Medium | Hypothesis (needs write-path trace) | `schema.ts:276` vs `:279` | Assert each report type's params validated on save | Apply `reportTypeSchema` on save — Ticket M9 |
| 12 | **Dual SQL branches can drift** — parallel Postgres/ClickHouse implementations, no parity test | Medium | Medium | Confirmed | e.g. `getWebsiteStats.ts` two branches | Parity test on a seeded dataset | Parity test (not forced abstraction) — Kata 4 |
| 13 | **Unbounded `pageSize` + double count** — list endpoints scan twice and allow huge pages | Medium | Medium | Confirmed | `schema.ts:79`, `prisma.ts:325-329` | Assert `pageSize` cap; measure count cost | Cap pageSize (Ticket 5); cursor pagination (Project S4) |
| 14 | **Global prototype patch** — `BigInt.prototype.toJSON`→Number, process-wide, precision-lossy >2^53 | Low | Low | Confirmed | `db.ts:8-10` | Assert large aggregate serializes; check precision | Scoped serializer instead of prototype patch |
| 15 | **Divergent entity reads in permissions** — `getEntity` vs `getWebsite` for the "same" concept | Low | Low | Confirmed | `website.ts:23` vs `:67` | — | Unify the read path |
| 16 | **No login rate limiting** — brute-force exposure | Medium | Medium | Confirmed | `login/route.ts` (no throttle) | N failures → throttled | IP-based login limiter — Ticket M8 |
| 17 | **Thin observability** — no metrics/tracing; failures are `console.log` only | Medium | High | Confirmed | `send/route.ts:318`, no metrics in repo | — | Metrics + reconciliation — Project S3 |
| 18 | **Stateless tokens can't be revoked** — logout only clears localStorage | Low-Med | Low | Confirmed | `login/route.ts:39` (non-Redis path) | — | Use Redis auth path where revocation matters |

## How to read this

- **Confirmed** items are verified in code (see the anchor). **Hypothesis** items are plausible from reading but need a runtime or write-path check.
- The top three (fire-and-forget delivery, per-handler authz, missing rate limiting) are the ones a senior would raise first in a review — they're the highest severity × reach.
- Every row maps to a ticket/project so the register is *actionable*, not just a list of complaints.

## For learners

- **Interview use:** pick any row and practice "here's a risk I found, here's why it matters, here's how I'd verify and fix it, and here's the tradeoff of the fix." That's a complete senior answer.
- **Contribution use:** rows 4, 5, 8, 9, 13, 16 are approachable as tickets; 1, 2, 3, 17 are senior projects.
- **Honesty note:** items are framed as risks/tradeoffs, not vulnerabilities. Several (6, 10, 12) are *acceptable* tradeoffs for an analytics product — the skill is judging *which* matter for *which* domain, not flagging everything as broken.
