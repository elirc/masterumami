# Refactor & Design Katas

Nine senior katas. Each is a focused exercise in judgment, with self-grading criteria. Unlike tickets, these have no single right answer — the grade is on your *reasoning and tradeoff articulation*, which is exactly what senior interviews test.

---

## Kata 1: Fix the boundary leak (handler composition)

**Task:** The pixel, link, and batch routes call `send`'s `POST` handler directly, coupling four routes to one request contract ([Flow 4](../01-codebase-cartography/05-key-flows.md#flow-4-pixel--link-tracking--handler-composition)). Refactor to a shared `collectEvent(payload, request)` function they all call.

**Do:** extract the core of `send`'s `POST` into a function; have `send`, `p`, `q`, `batch` call it; keep the HTTP handlers thin.

**Self-grade:**
- *Basic:* you extracted the function and it compiles.
- *Solid:* the four callers use it; you preserved the exact response contract for each (GIF, redirect, JSON, batch aggregation).
- *Strong:* you identified what *must* stay in each handler (response shaping) vs. move (validation + persistence), and you added a test proving the public `/api/send` behavior is unchanged. You noted the blast radius (the public contract) and why this refactor doesn't change it.

---

## Kata 2: Design the outbox change (eventing)

**Task:** Write the design doc for at-least-once event delivery (Project S1) — no code, just the doc.

**Self-grade:**
- *Solid:* you specified the outbox table, the relay, and the write-before-publish ordering.
- *Strong:* you addressed duplicate delivery (idempotent ids), the shadow-mode migration, reconciliation, the added hot-path latency, and rollback. You stated what new failure mode the fix introduces (duplicates) and how it's absorbed.

---

## Kata 3: Split a module (`lib/prisma.ts`)

**Task:** `lib/prisma.ts` mixes the client singleton, the SQL-template engine, filter parsing, and pagination (437 lines). Propose a split into cohesive modules.

**Self-grade:**
- *Basic:* you proposed some file boundaries.
- *Solid:* your split follows single-responsibility (e.g. `sql-engine.ts`, `filters.ts`, `pagination.ts`, `client.ts`) and you kept the public `default` export stable.
- *Strong:* you sequenced the split to keep every step green (move + re-export, then update imports), sized the blast radius (every query file imports from here), and argued why *not* over-splitting matters too.

---

## Kata 4: Remove duplication safely (dual SQL branches)

**Task:** Every query file has near-parallel Postgres and ClickHouse implementations that can drift ([Pattern 1 failure mode](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-1-polymorphic-persistence-via-dispatch-table)). Propose how to reduce drift *without* a leaky abstraction.

**Self-grade:**
- *Solid:* you proposed a mechanism (shared SQL fragments, a parity test comparing results on a seeded dataset, or a codegen approach) and weighed it.
- *Strong:* you argued *against* over-abstracting (the two dialects genuinely differ — session denormalization on ClickHouse), and landed on a *parity test* as the pragmatic guard rather than a forced shared abstraction. Recognizing when *not* to DRY is the senior signal.

---

## Kata 5: Improve type safety (raw-SQL results)

**Task:** `rawQuery` returns `any` ([`prisma.ts:255`](../../../src/lib/prisma.ts)); result interfaces are hand-maintained and unchecked. Propose a way to increase safety.

**Self-grade:**
- *Solid:* you proposed typing `rawQuery<T>` and annotating call sites.
- *Strong:* you acknowledged that generics don't *verify* the SQL matches the type (it's still a promise, not a proof), proposed a runtime check (Zod-parse the first row in dev, or a test asserting columns), and used `satisfies` where casts exist. You correctly scoped this as risk-reduction, not elimination.

---

## Kata 6: Design a migration (add a column to both stores)

**Task:** Add a `bot_score` field to events, populated at collection, queryable in reports — across Postgres *and* ClickHouse.

**Self-grade:**
- *Solid:* you listed the Prisma migration, the ClickHouse schema change, both `saveEvent` branches, and the report query.
- *Strong:* you made it additive/nullable (safe rollback), sequenced expand→migrate→contract, addressed backfill for existing rows, respected the public payload contract, and noted the trap that a Prisma-only change breaks ClickHouse deployments.

---

## Kata 7: Reduce an N+1 (or prove there isn't one)

**Task:** Audit a dashboard view that renders multiple websites with per-website stats. Determine whether it N+1s, and fix or justify.

**Self-grade:**
- *Solid:* you used `LOG_QUERY` reasoning to count queries and identified whether it's one-per-website.
- *Strong:* if N+1, you proposed a single grouped query keyed by website id; if not, you *proved* it (the query already groups) rather than assuming. You measured rather than guessed.

---

## Kata 8: Write an RFC (choose one real change)

**Task:** Write a one-page RFC for one mid/senior change (e.g. rate limiting, the outbox, cursor pagination) using the RFC template in [`../07-career-and-collaboration/02-writing-prs-and-rfcs.md`](../07-career-and-collaboration/02-writing-prs-and-rfcs.md).

**Self-grade:**
- *Solid:* problem, proposal, alternatives, risks.
- *Strong:* you included a migration/rollback plan, a test strategy, explicit non-goals, and open questions — and you named at least one alternative you *rejected* and why. An RFC that only presents the chosen option reads as junior.

---

## Kata 9: Review a flawed PR (self-authored)

**Task:** Deliberately write a plausible-but-flawed diff for a real Umami change (e.g. "cache the record website lookup"), then review your own PR as a maintainer would.

**Self-grade:**
- *Solid:* you found the primary flaw.
- *Strong:* you graded findings blocking/important/optional, wrote kind+specific comments, and caught a non-obvious consequence (e.g. the 24h staleness regression). This mirrors [Review Kata 3](../04-code-reading-gym/04-review-katas.md).

---

## The meta-skill

Every kata rewards the same thing: **articulating tradeoffs and blast radius, and knowing when *not* to do the clever thing** (Kata 4's "don't over-abstract," Kata 3's "don't over-split"). Seniors are hired for judgment, not for knowing the most patterns. Practice narrating your reasoning aloud — that narration *is* the senior interview. Feed the results into your STAR stories ([`../08-interview-prep/06-behavioral-star-stories.md`](../08-interview-prep/06-behavioral-star-stories.md)).
