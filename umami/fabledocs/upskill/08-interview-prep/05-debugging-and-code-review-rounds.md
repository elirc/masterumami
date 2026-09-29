# Interview Prep: Debugging & Code Review Rounds

These live rounds test how you *think*, not what you memorized. This page turns Umami's debugging scenarios and review katas into timed simulations with interviewer follow-ups and grading rubrics. Practice narrating aloud — the narration is what's graded.

## How these rounds work

- **Debugging round:** you're given a symptom (sometimes a repo to poke). You reproduce, narrow, hypothesize, and fix — talking the whole time. Interviewers interrupt with "why do you think that?" and "what else could it be?"
- **Review round:** you're given a diff and asked to review it. You find issues, grade severity, and phrase feedback kindly.

The universal tell of a strong candidate: **you split the problem space before guessing.** "Is this write-side or read-side?" before touching anything.

---

## Debugging Simulation 1 — Silent event loss (10 min)

**Prompt (interviewer reads):** "Customers on our large-scale deployment say some events don't show up, but the tracker reports success. Where do you start?"

**Your narration should hit:**
1. "First, separate write-side vs read-side — are the events in the store at all?" (If not persisted → write-side.)
2. "I'd check the deployment: large-scale means ClickHouse + Kafka. Let me look at the write path." → `saveEvent` ClickHouse branch → `kafka.sendMessage`.
3. "Here — `sendMessage` catches and only `console.log`s ([kafka.ts:87-90](../../src/lib/kafka.ts)), returning undefined. So a broker failure is swallowed and `/api/send` still returns 200. That's at-most-once delivery."
4. "I'd confirm by grepping logs for `KAFKA ERROR` co-occurring with 200s."

**Interviewer follow-ups & good answers:**
- *"How would you fix it?"* → "Not a naive retry — an outbox: durably persist on accept, relay to Kafka with retries. Duplicates are safe because session ids are deterministic."
- *"What new problem does that create?"* → "Duplicate delivery, absorbed by idempotent keys / `on conflict`."
- *"How would you have caught this earlier?"* → "A reconciliation metric: events accepted vs. persisted."

**Rubric:** *Basic* — finds the swallowed catch. *Solid* — names at-most-once and proposes an outbox. *Strong* — handles the duplicate consequence, adds observability, and split write/read first. (Full version: [Debugging Scenario 2](../05-quality-engineering/03-systematic-debugging.md#scenario-2-some-events-return-200-but-never-appear-in-the-dashboard).)

---

## Debugging Simulation 2 — Cross-tenant data leak (10 min)

**Prompt:** "A user reports seeing another company's website in an API response. Walk me through it."

**Narration:**
1. "AuthN or authZ? Is the user actually authenticated as themselves?" → confirm token resolves to them → it's authorization.
2. "I'd compare the leaking route to a correct one. Correct routes call `canViewWebsite(auth, id)` after `parseRequest` ([realtime route:20](../../src/app/api/realtime/[websiteId]/route.ts))."
3. "The leaking route is missing that check, or checks the wrong id — a classic IDOR."
4. "Fix: add the guard. Then the bigger question — how do we ensure *every* route does it?"

**Follow-ups:**
- *"How do you prevent the whole class?"* → a `withAuthorization` wrapper or a lint/test enumerating `[id]` routes.
- *"How do you test it?"* → cross-tenant request asserting 401, added to e2e.

**Rubric:** *Solid* separates authN from authZ and finds the missing check. *Strong* proposes the systemic guardrail and the regression test. (Full: [Scenario 3](../05-quality-engineering/03-systematic-debugging.md#scenario-3-a-user-can-see-another-users-website-metrics).)

---

## Debugging Simulation 3 — Slow list endpoint (10 min)

**Prompt:** "The sessions page times out for a big customer. Fix it."

**Narration:**
1. "DB or app? I'd bet DB. `LOG_QUERY=1` and read the SQL for `getWebsiteSessions`."
2. "`EXPLAIN ANALYZE` it — is it using the composite index or seq-scanning? The indexes exist ([schema.prisma:139-148](../../prisma/schema.prisma)), so I suspect the *inputs*."
3. "Two suspects: unbounded date range, and `pagedRawQuery` runs a `count(*)` plus the page query ([prisma.ts:325](../../src/lib/prisma.ts)) — two scans. Plus `pageSize` has no cap ([schema.ts:79](../../src/lib/schema.ts))."
4. "Fix: bound pageSize; consider cursor pagination to drop the count; sane date-range default. Re-measure."

**Follow-ups:**
- *"Would you add an index?"* → "Not first — measure. The indexes are likely fine; the cost is unbounded inputs and a redundant count. Adding indexes also slows this write-heavy table."
- *"Cursor vs offset?"* → keyset avoids the count and deep-offset cost; loses random page access.

**Rubric:** *Strong* resists the reflexive "add an index," measures first, and identifies the count + unbounded inputs. (Full: [Scenario 4](../05-quality-engineering/03-systematic-debugging.md#scenario-4-the-sessions-list-is-slow-and-sometimes-times-out-for-big-sites).)

---

## Review Simulation 1 — "Add sorting" PR (8 min)

**Prompt:** "Review this PR that adds user-selectable sorting to the sessions list by passing `orderBy` through to `pagedRawQuery`."

**Expected review (say it aloud, graded):**
- **Blocking:** `pagedRawQuery` interpolates `order by ${orderBy}` ([prisma.ts:318](../../src/lib/prisma.ts)) — user input here is SQL injection. Require an allow-list.
- **Important:** interaction with the existing hardcoded `order by ... desc`.
- **Optional:** test the rejection path.

**Good phrasing:** *"Blocking, but easy fix: `orderBy` is string-interpolated into the SQL, so this is an injection vector. Can we map the requested column through an allow-list and reject anything else? Happy to pair."*

**Follow-up:** *"Show me the allow-list."* → `{ createdAt: 'created_at', ... }`, reject misses.

**Rubric:** *Strong* catches the injection, grades it blocking, phrases it kindly, and provides the fix. (Full: [Review Kata 1](../04-code-reading-gym/04-review-katas.md#kata-1-add-sorting-to-the-sessions-list).)

---

## Review Simulation 2 — "Cache the record lookup" PR (8 min)

**Prompt:** "Review this PR that changes `getWebsite` to the cached `fetchWebsite` in the record endpoint, to reduce DB load."

**Expected review:**
- **Blocking:** the direct read is deliberate (comment at [record/route.ts:56](../../src/app/api/record/route.ts)) — `replayEnabled` must be fresh; caching reintroduces up-to-24h staleness, so disabling replay wouldn't take effect. Regression of a correctness choice.
- **Optional:** if caching is truly wanted, pair it with invalidation on update.

**Good phrasing:** *"This bypass is intentional — the comment explains replay must be read fresh. `fetchWebsite` caches 24h, so toggling replay off would lag up to a day. I'd keep the direct read unless we also invalidate on update."*

**Rubric:** *Strong* recognizes the "fix" is a regression and cites the comment. (Full: [Review Kata 3](../04-code-reading-gym/04-review-katas.md#kata-3-cache-the-website-lookup-in-the-record-endpoint).)

---

## How to practice these

- **Timebox** each simulation. Debugging: 10 min. Review: 8 min. Interviews are timed; practice the clock.
- **Narrate continuously.** Silence reads as being stuck. "I'm now checking whether…" keeps the interviewer with you.
- **Always split the space first** ("write vs read," "authN vs authZ," "DB vs app") before guessing.
- **End with prevention.** "…and I'd add a regression test / metric so it can't recur" is the closer that reads senior.
- **Record yourself** once. You'll hear the filler and the places you jumped to a fix without narrowing.

## Self-grading rubric (all rounds)

| Level | Debugging | Review |
| --- | --- | --- |
| Basic | finds *a* cause eventually | finds the main issue |
| Solid | splits the space, narrows methodically, fixes the root cause | grades severity, phrases kindly, cites the line |
| Strong | + adds a regression test/metric, names what else it could've been, resists reflexive fixes | + catches non-obvious consequences, knows when to approve, offers the fix |

Do two debugging and two review simulations in your final cram week ([`07-two-week-cram-plan.md`](07-two-week-cram-plan.md)).
