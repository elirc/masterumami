# 08 — Interview Prep

A first-class module, not an afterthought. Target: **mid-level fullstack JS interviews**. At least 60% of the question cards are anchored to real Umami code, so you practice answering with concrete evidence — "in a codebase I studied, this was implemented as… and it fails when…" — instead of reciting definitions.

Files:

1. [`01-js-ts-node-deep-dive.md`](01-js-ts-node-deep-dive.md) — 14 cards on the event loop, async, closures, TS types, Node runtime.
2. [`02-frontend-framework-questions.md`](02-frontend-framework-questions.md) — 12 cards on React 19, state, React Query, rendering, a11y.
3. [`03-api-and-data-modeling-questions.md`](03-api-and-data-modeling-questions.md) — 12 cards on REST design, validation, pagination, auth, schema, transactions, caching.
4. [`04-system-design-from-this-repo.md`](04-system-design-from-this-repo.md) — the full "design a web analytics platform" walkthrough + 5 variations.
5. [`05-debugging-and-code-review-rounds.md`](05-debugging-and-code-review-rounds.md) — timed debugging + review simulations with interviewer follow-ups.
6. [`06-behavioral-star-stories.md`](06-behavioral-star-stories.md) — 10 STAR worksheets sourced from this repo's tickets/projects.
7. [`07-two-week-cram-plan.md`](07-two-week-cram-plan.md) — a day-by-day plan tying it all together.

## How mid-level fullstack loops are structured

A typical loop:
1. **Recruiter/screen** — motivation, basics, one or two easy technical questions.
2. **Technical phone screen** — a coding problem + language depth (module 08-01).
3. **Practical coding** — build/extend something small (the tickets in module 06 are practice).
4. **System design** — often lighter for mid-level, but present (08-04).
5. **Debugging / code review** — increasingly common; read/fix real code (08-05).
6. **Behavioral** — collaboration, conflict, ownership (08-06).

## The golden rule

Every answer = **concrete example + tradeoff + failure mode.** Not a definition. "A cache-aside pattern is…" is a junior answer. "In the analytics codebase I studied, website lookups were cache-aside with a `__DELETED__` tombstone to prevent a delete/repopulate race, but the 24h TTL meant some flags went stale — so one endpoint deliberately bypassed the cache" is a mid/senior answer. The whole point of anchoring to Umami is to give you that second kind of answer for dozens of topics.

## How to use this repo as your portfolio of talking points

You studied a real, non-trivial system. That's your material. When asked *any* concept question, reach for how Umami did it, what it cost, and how it fails. Two weeks of doing that turns "I read a tutorial" into "I've reasoned about a production system," which is exactly the level jump interviewers are grading.

Start with the [two-week cram plan](07-two-week-cram-plan.md) if you're on the clock; otherwise work the cards in order.
