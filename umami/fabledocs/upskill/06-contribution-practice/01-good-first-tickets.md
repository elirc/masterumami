# Good First Tickets

Twenty junior tickets spread across the repo — collection, query layer, permissions, tracker, tests, and client. Each has a small blast radius, follows an existing pattern, and is testable. Every ticket names the pattern it follows and the STAR story completing it would produce.

**Contribution standard (applies to all):** follows an existing pattern, small blast radius, has a test, would not be rejected for touching unrelated code. Read [`07-career-and-collaboration/02-writing-prs-and-rfcs.md`](../07-career-and-collaboration/02-writing-prs-and-rfcs.md) before opening a PR.

Difficulty legend: Easy = 1 file, <1h. Medium = 2–3 files, 1–3h.

---

## Ticket 1: Unit-test `hasBlockedIp` CIDR matching

**Difficulty:** Easy — 30–45 min. **Skills:** unit testing, reading pure functions.

**Story:** As a maintainer, I want `hasBlockedIp` covered so IP-blocking behavior can't silently regress.

**Why good:** Pure function, no DB, high-value security path, currently untested.

**Acceptance criteria:**
- [ ] Test exact-match block, CIDR-range block, and non-match pass-through.
- [ ] Sets/clears `process.env.IGNORE_IP` per case.
- [ ] `pnpm test` green.

**Read first:** [`src/lib/detect.ts:140-170`](../../../src/lib/detect.ts); mirror style of [`src/lib/detect.test.ts`](../../../src/lib/detect.test.ts).

**Files touched:** `src/lib/detect.test.ts` (extend).

**Plan:** Set `IGNORE_IP='10.0.0.5,192.168.1.0/24'`; assert `hasBlockedIp('10.0.0.5')` truthy, `hasBlockedIp('192.168.1.50')` truthy (CIDR), `hasBlockedIp('8.8.8.8')` falsy.

**What could go wrong:** `ipaddr.parse` throws on malformed input — test that path too.

**Review questions:** Did you restore env vars after each test?

**Interview story potential:** "I added regression coverage to an untested security filter and found its behavior on malformed IPs was undefined."

---

## Ticket 2: Add a test asserting event-name truncation

**Difficulty:** Easy — 45 min. **Skills:** testing, understanding write-path constraints.

**Story:** As a maintainer, I want proof that event names over 50 chars are truncated before insert.

**Why good:** Documents an invariant (`EVENT_NAME_LENGTH`) that protects the DB `varchar(50)` column.

**Acceptance criteria:**
- [ ] Test that a 60-char name is stored as 50 chars.
- [ ] Uses a mock/spy on the prisma client rather than a real DB.

**Read first:** [`saveEvent.ts:131`](../../../src/queries/sql/events/saveEvent.ts), [`constants.ts:257`](../../../src/lib/constants.ts).

**Files touched:** new `src/queries/sql/events/saveEvent.test.ts`.

**Plan:** Mock `@/lib/prisma`, call the relational branch with a long name, assert the `create` call's `eventName.length === 50`.

**What could go wrong:** Mocking the dispatch (`runQuery`) is fiddly — you may need to force the Prisma branch by clearing `CLICKHOUSE_URL`.

**Interview story potential:** "I wrote a test that pinned a truncation invariant matching a DB column width."

---

## Ticket 3: Extract the `'https://localhost'` base URL into a named constant

**Difficulty:** Easy — 30 min. **Skills:** removing duplication.

**Story:** As a maintainer, I want the synthetic base URL defined once.

**Why good:** The literal appears 3× in `send/route.ts` ([`:178`, `:283`](../../../src/app/api/send/route.ts)) and in `schema.ts:95`. Follows the constants pattern.

**Acceptance criteria:**
- [ ] One exported constant; all uses reference it.
- [ ] No behavior change; typecheck clean.

**Read first:** the three call sites; `src/lib/constants.ts`.

**What could go wrong:** Don't change the `/undefined` handling logic while you're in there — scope creep.

**Interview story potential:** "Small DRY cleanup that made a magic string searchable."

---

## Ticket 4: Add `describe` grouping and an edge case to `boards.test.ts`

**Difficulty:** Easy — 30 min. **Skills:** test organization.

**Story:** Group the flat `test()`s and add a case for an unknown component name.

**Read first:** [`src/lib/boards.test.ts`](../../../src/lib/boards.test.ts), [`src/lib/boards.ts`](../../../src/lib/boards.ts).

**Acceptance criteria:** New case asserts `isBoardComponentSupported('Nonexistent', ...)` behavior; tests still pass.

**What could go wrong:** Confirm the *actual* return for unknown components before asserting — don't guess.

**Interview story potential:** "Hardened a compatibility-matrix test with the undefined-input case."

---

## Ticket 5: Validate `pageSize` upper bound in `pagingParams`

**Difficulty:** Medium — 1–2h. **Skills:** validation, denial-of-service thinking.

**Story:** As a maintainer, I want to reject absurd page sizes so a client can't request 10M rows.

**Why good:** `pagingParams.pageSize` is `positive()` with no max ([`schema.ts:79`](../../../src/lib/schema.ts)); unbounded page size is a real query-cost risk.

**Acceptance criteria:**
- [ ] `pageSize` capped (e.g. `.max(1000)`).
- [ ] A test asserts oversized values are rejected.
- [ ] Confirm no existing route legitimately needs >cap.

**Read first:** `schema.ts:77-80`; how `pagedRawQuery` uses size ([`prisma.ts:313-316`](../../../src/lib/prisma.ts)).

**What could go wrong:** Some internal caller might pass a large size on purpose — grep usages first. Choose the cap with evidence.

**Interview story potential:** "I bounded an unbounded pagination param and reasoned about the query-cost blast radius before picking the limit."

---

## Ticket 6: Add an allow-list guard comment/TODO at the `orderBy` interpolation

**Difficulty:** Easy — 30 min. **Skills:** security awareness, honest documentation.

**Story:** As a maintainer, I want the latent injection surface at `order by ${orderBy}` clearly marked so no one wires user input into it.

**Why good:** Documents a real risk (see risk register) without changing behavior — the safest possible first step.

**Acceptance criteria:**
- [ ] A comment at [`prisma.ts:318-323`](../../../src/lib/prisma.ts) stating `orderBy` must be from a trusted allow-list, never raw user input.
- [ ] No code change.

**Read first:** the risk register entry; confirm current callers don't pass user `orderBy`.

**What could go wrong:** Overstating it as a live bug — label it "must remain allow-listed," not "is vulnerable."

**Interview story potential:** "I found a latent SQL-injection surface, confirmed it wasn't currently reachable, and documented the invariant that keeps it safe."

---

## Ticket 7: Unit-test `flattenJSON` / `getStringValue`

**Difficulty:** Medium — 1–2h. **Skills:** testing data transforms.

**Story:** Cover the JSON-to-typed-rows flattening used by every custom event.

**Read first:** [`src/lib/data.ts`](../../../src/lib/data.ts) (`flattenJSON`, `getStringValue`), used at [`saveEventData.ts:30-43`](../../../src/queries/sql/events/saveEventData.ts).

**Acceptance criteria:** Tests for nested objects, arrays, numbers, dates, and how each maps to `dataType`.

**What could go wrong:** Date detection may be heuristic — read the actual logic before asserting.

**Interview story potential:** "I characterized an under-tested EAV flattening function and pinned its type inference."

---

## Ticket 8: Add `enabled`-guard consistency to a query hook missing it

**Difficulty:** Easy — 30 min. **Skills:** React Query correctness.

**Story:** Ensure hooks that need a `websiteId` set `enabled: !!websiteId` so they don't fire with `undefined`.

**Read first:** the pattern in [`useWebsiteMetricsQuery.ts:39`](../../../src/components/hooks/queries/useWebsiteMetricsQuery.ts); audit siblings in `src/components/hooks/queries/`.

**Acceptance criteria:** Identify one hook missing the guard (if any), add it; note in the PR which you audited.

**What could go wrong:** If all hooks already have it, the ticket becomes "document that they do" — still valuable. Don't invent a fix.

**Interview story potential:** "Audited 60 data hooks for a defensive guard and standardized the outliers."

---

## Ticket 9: Reject empty `events` array earlier in `/api/record`

**Difficulty:** Easy — 30 min. **Skills:** input validation.

**Story:** The schema allows `events: []`; the handler then special-cases it ([`record/route.ts:37-39`](../../../src/app/api/record/route.ts)). Move the guard into Zod (`.min(1)`).

**Read first:** `record/route.ts:18-39`.

**Acceptance criteria:** Schema requires ≥1 event; handler special-case removed; behavior for empty is now a 400.

**What could go wrong:** A legitimate client might send empty batches as heartbeats — check the recorder before changing the contract. If so, keep the current behavior and document why.

**Interview story potential:** "I pushed a runtime guard up into the schema, then discovered a client depended on the lenient behavior — a lesson in checking callers first."

---

## Ticket 10: Add a `max()` bound to the batch array size

**Difficulty:** Easy — 45 min. **Skills:** DoS awareness.

**Story:** `/api/batch` accepts `z.array(anyObjectParam)` with no max ([`batch/route.ts:7`](../../../src/app/api/batch/route.ts)); each item re-enters the full send handler sequentially.

**Read first:** `batch/route.ts`; compare `record`'s `.max(200)` cap.

**Acceptance criteria:** Array capped (choose a defensible number, justify it); test rejects over-cap.

**What could go wrong:** Pick the cap from evidence (what does the tracker actually batch?), not a guess.

**Interview story potential:** "I bounded an unbounded batch endpoint to cut its DoS amplification."

---

## Ticket 11: Document env vars consumed by `next.config.ts`

**Difficulty:** Easy — 1h. **Skills:** operational docs.

**Story:** List the ~15 env vars read in `next.config.ts` with one-line descriptions.

**Read first:** [`next.config.ts`](../../../next.config.ts) top section.

**Acceptance criteria:** A table in `fabledocs/` or a repo doc; no secrets, just names and purposes.

**Interview story potential:** "I mapped the deployment configuration surface for onboarding."

---

## Ticket 12: Test `getDevice` laptop-vs-desktop heuristic

**Difficulty:** Easy — 45 min. **Skills:** testing branching logic.

**Story:** `getDevice` calls a desktop ≤1920px wide a "laptop" ([`detect.ts:49-61`](../../../src/lib/detect.ts)). Pin this with tests.

**Acceptance criteria:** Cases for mobile UA, desktop UA with wide screen, desktop UA with `≤1920` screen → "laptop".

**What could go wrong:** UA parsing depends on `ua-parser-js`; use realistic UA strings.

**Interview story potential:** "I pinned a surprising device-classification heuristic before anyone refactored it blind."

---

## Ticket 13: Add JSDoc to `runQuery` naming the three backends

**Difficulty:** Easy — 20 min. **Skills:** documenting a core abstraction.

**Story:** `runQuery` is the persistence dispatcher; a one-block comment saves every new reader a spelunk.

**Read first:** [`db.ts:22-36`](../../../src/lib/db.ts).

**Acceptance criteria:** JSDoc listing the Kafka→ClickHouse→Prisma selection order and the env vars that drive it.

**Interview story potential:** "Documented the load-bearing dispatch function everyone reads first."

---

## Ticket 14: Add a `.max()` to `payload.title` in the send schema

**Difficulty:** Easy — 30 min. **Skills:** validation/DB-alignment.

**Story:** `title` has no length cap in the schema ([`send/route.ts:36`](../../../src/app/api/send/route.ts)) though it's truncated to `PAGE_TITLE_LENGTH` at write. Align the schema with the column.

**Read first:** `send/route.ts:36`, `saveEvent.ts:123`, `constants.ts:256`.

**Acceptance criteria:** Schema caps `title` at `PAGE_TITLE_LENGTH`; a test asserts rejection over cap.

**What could go wrong:** Truncation vs. rejection are different UX choices — match the existing convention (other fields cap in schema).

**Interview story potential:** "I aligned a validation bound with its database column to fail fast at the boundary."

---

## Ticket 15: Unit-test the `visitId` 30-minute expiry logic

**Difficulty:** Medium — 2h. **Skills:** testing time-dependent logic.

**Story:** Cover the "expire visit after 30 minutes" branch ([`send/route.ts:171-175`](../../../src/app/api/send/route.ts)).

**Read first:** the block; note it uses `Math.floor(Date.now()/1000)`.

**Acceptance criteria:** Extract the pure part if needed; test that `now - iat > 1800` yields a new `visitId`.

**What could go wrong:** The logic is inline in the handler — this may require a small extraction (get maintainer buy-in first; it edges toward mid-level).

**Interview story potential:** "I made time-based session logic testable by extracting a pure helper."

---

## Ticket 16: Add `data-test` attributes to an untested settings form

**Difficulty:** Medium — 1–2h. **Skills:** enabling e2e testing.

**Story:** Cypress selects by `data-test` ([`login.cy.ts`](../../../cypress/e2e/login.cy.ts) uses `getDataTest`). Add attributes to a form that lacks them to unblock future e2e.

**Read first:** `LoginForm.tsx` for the convention; pick a form under `src/app/(main)/.../settings`.

**Acceptance criteria:** Attributes added following the `input-*`/`button-*` naming; no logic change.

**Interview story potential:** "I made a form testable and wrote the first e2e for it."

---

## Ticket 17: Guard `getSalt` against an unknown rotation value

**Difficulty:** Easy — 45 min. **Skills:** defensive defaults.

**Story:** `getSalt` treats anything not `day`/`week` as `month` ([`crypto.ts:72-78`](../../../src/lib/crypto.ts)). Add a test documenting this and a comment.

**Acceptance criteria:** Test asserts `getSalt('garbage', d)` behaves like `month`.

**What could go wrong:** This is documenting current behavior, not changing it — resist "fixing" the fall-through.

**Interview story potential:** "I pinned a silent-default behavior so a future change to salt rotation can't surprise anyone."

---

## Ticket 18: Add a health-detail field to `/api/heartbeat`

**Difficulty:** Medium — 1–2h. **Skills:** observability, reading existing routes.

**Story:** The heartbeat is used by the docker healthcheck ([`docker-compose.yml`](../../../docker-compose.yml)). Confirm what it returns and, if trivial-only, add a version field.

**Read first:** `src/app/api/heartbeat/route.ts`.

**Acceptance criteria:** Response includes app version from `package.json`; healthcheck still passes.

**What could go wrong:** Don't add a DB ping that could make the healthcheck flap — keep it cheap.

**Interview story potential:** "I enriched a health endpoint without making it a false-alarm source."

---

## Ticket 19: Normalize `referer`/`referrer` header spelling

**Difficulty:** Easy — 30 min. **Skills:** attention to detail.

**Story:** The pixel/link routes read `'referer'` ([`p/[slug]/route.ts:50`](../../../src/app/(collect)/p/[slug]/route.ts)) — the HTTP misspelling. Confirm it's correct (it is, per spec) and add a clarifying comment so no one "fixes" it to `referrer` and breaks it.

**Acceptance criteria:** Comment added; no behavior change.

**What could go wrong:** This is the opposite of a fix — the misspelling is the *correct* header name. Document that.

**Interview story potential:** "I prevented a plausible-looking 'typo fix' that would have broken referrer tracking."

---

## Ticket 20: Add a test for the `parseToken` failure path

**Difficulty:** Easy — 45 min. **Skills:** testing error handling.

**Story:** `parseToken`/`parseSecureToken` return `null` on invalid input ([`jwt.ts:8-26`](../../../src/lib/jwt.ts)). Pin that they never throw.

**Acceptance criteria:** Tests pass a malformed token, an expired token, and a wrong-secret token; all return `null`.

**What could go wrong:** Generating an expired token needs a signed token with a past `exp` — use `jsonwebtoken` options.

**Interview story potential:** "I proved our token parsers fail closed and never throw into the request pipeline."

---

## Picking your first ticket

Start with **1, 4, or 13** (pure, no DB, under an hour). Then do **5 or 10** to practice reasoning about a bound before choosing it — that "justify the number with evidence" habit is exactly what separates a mid-level PR from a drive-by. Each ticket's "Interview story potential" line feeds directly into [`08-interview-prep/06-behavioral-star-stories.md`](../08-interview-prep/06-behavioral-star-stories.md).
