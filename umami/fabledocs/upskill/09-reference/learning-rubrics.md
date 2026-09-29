# Learning Rubrics

Self-assessment with **observable behaviors**, not vague traits. For each skill: what junior/mid/senior *does* (not "understands"), plus an **interview-ready** column — the behavior that reads as mid-level in an interview. Grade yourself honestly by what you can *do right now*, using Umami as the reference system.

Rule: you can only claim a level if you can *demonstrate* the behavior against real code — trace it, change it, or explain it aloud with anchors.

## How to use

Rate yourself on each row. Anywhere you're below "interview-ready," the linked module is your gap-filler. Re-rate at the day-7 and day-14 checkpoints ([`../08-interview-prep/07-two-week-cram-plan.md`](../08-interview-prep/07-two-week-cram-plan.md)).

---

## Skill: Reading unfamiliar code

| Level | Observable behavior |
| --- | --- |
| Junior | Can locate where a feature lives with search; reads a file top to bottom. |
| Mid | Traces a value across files; names a function's inputs/outputs/side effects without running it. |
| Senior | Names invariants, failure modes, and hidden coupling on first read; predicts blast radius of a change. |
| **Interview-ready** | Given a file cold, narrates what it does, what could go wrong, and one risk — aloud, in 2 minutes. |

Gap-filler: [`../04-code-reading-gym/`](../04-code-reading-gym/README.md).

## Skill: The event loop & async

| Level | Behavior |
| --- | --- |
| Junior | Knows `await` "waits." |
| Mid | Explains that `await` yields the thread; distinguishes concurrency from parallelism; spots needless serial awaits (`saveEvent.ts:143-167`). |
| Senior | Reasons about microtask vs macrotask ordering; identifies when to `Promise.all` vs keep serial for correctness. |
| **Interview-ready** | Walks the event loop through a real async handler and names one place to parallelize and one that must stay serial. |

Gap-filler: [`../02-stack-and-language-mastery/01-language-runtime-model.md`](../02-stack-and-language-mastery/01-language-runtime-model.md).

## Skill: TypeScript & contracts

| Level | Behavior |
| --- | --- |
| Junior | Adds type annotations; uses `any` when stuck. |
| Mid | Validates at boundaries (Zod), trusts types inside; knows `unknown` vs `any`; reads discriminated unions. |
| Senior | Identifies where `any` is an acceptable escape hatch vs a smell; knows the raw-SQL type-safety gap and mitigates it. |
| **Interview-ready** | Explains "where's your validation boundary and why validate if you have types?" with the Zod/`parseRequest` example. |

Gap-filler: [`../02-stack-and-language-mastery/03-type-system-and-contracts.md`](../02-stack-and-language-mastery/03-type-system-and-contracts.md).

## Skill: React & state

| Level | Behavior |
| --- | --- |
| Junior | Builds components; uses `useState`; fetches in effects. |
| Mid | Categorizes state (server/global/local) and picks the right tool; writes correct React Query keys; knows render vs commit. |
| Senior | Diagnoses unnecessary re-renders; reasons about the server/client boundary's bundle cost; knows React 19 compiler implications. |
| **Interview-ready** | Explains the three kinds of state with Umami's choices and names the bug from an incomplete `queryKey`. |

Gap-filler: [`../02-stack-and-language-mastery/02-framework-mental-models.md`](../02-stack-and-language-mastery/02-framework-mental-models.md).

## Skill: API design

| Level | Behavior |
| --- | --- |
| Junior | Writes CRUD routes; returns JSON. |
| Mid | Applies validate→authN→authZ; designs pagination; evolves a contract backward-compatibly. |
| Senior | Defends "unauthenticated on purpose"; picks pagination strategy by tenant size; reasons about public-contract permanence. |
| **Interview-ready** | Designs the collection endpoint aloud, defends the auth choice, and names the abuse mitigations. |

Gap-filler: [`../08-interview-prep/03-api-and-data-modeling-questions.md`](../08-interview-prep/03-api-and-data-modeling-questions.md).

## Skill: Data modeling

| Level | Behavior |
| --- | --- |
| Junior | Creates tables with obvious columns; adds indexes on filtered columns. |
| Mid | Chooses normalization vs denormalization by access pattern; designs composite indexes leading with tenant+time; knows EAV vs JSONB. |
| Senior | Plans safe schema migrations (additive, expand/contract); reasons about dual-store consistency. |
| **Interview-ready** | Designs the events table + indexes and explains why the index leads with `(website_id, created_at, X)`. |

Gap-filler: [`../03-architecture-and-patterns/02-data-model-and-persistence.md`](../03-architecture-and-patterns/02-data-model-and-persistence.md).

## Skill: Authorization & security

| Level | Behavior |
| --- | --- |
| Junior | Checks the user is logged in. |
| Mid | Separates authN from authZ; adds per-resource `can*` checks; recognizes IDOR; validates input. |
| Senior | Sees the systemic risk of per-handler authz; distinguishes capability vs instance authorization; reasons about trust boundaries and injection surfaces. |
| **Interview-ready** | Tries a cross-tenant request to test IDOR and proposes a systemic guardrail, not just a per-route fix. |

Gap-filler: [`../03-architecture-and-patterns/03-validation-auth-and-permissions.md`](../03-architecture-and-patterns/03-validation-auth-and-permissions.md), [`../05-quality-engineering/05-security-checklist.md`](../05-quality-engineering/05-security-checklist.md).

## Skill: Reliability & side effects

| Level | Behavior |
| --- | --- |
| Junior | Writes to the DB; assumes it works. |
| Mid | Recognizes fire-and-forget risk; knows idempotency via deterministic keys; spots missing transactions. |
| Senior | Designs at-least-once delivery with an outbox; reasons about duplicates and reconciliation; judges which atomicity gaps a domain tolerates. |
| **Interview-ready** | Explains the Kafka silent-loss gap, proposes an outbox, and names the duplicate consequence it creates. |

Gap-filler: [`../03-architecture-and-patterns/04-side-effects-async-and-reliability.md`](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md).

## Skill: Debugging

| Level | Behavior |
| --- | --- |
| Junior | Tries fixes at the first plausible line. |
| Mid | Reproduces, splits the problem space, forms one hypothesis, tests it cheaply, fixes the root cause. |
| Senior | Adds a regression test before closing; names what else it could have been; resists reflexive fixes (e.g. "add an index"). |
| **Interview-ready** | Narrates a debugging round splitting write/read (or authN/authZ) first, ending with a regression test. |

Gap-filler: [`../05-quality-engineering/03-systematic-debugging.md`](../05-quality-engineering/03-systematic-debugging.md).

## Skill: Testing

| Level | Behavior |
| --- | --- |
| Junior | Writes happy-path tests; tests implementation details. |
| Mid | Picks the right layer; tests behavior/contracts; handles time/randomness; tests the deny path. |
| Senior | Knows what *not* to test; designs for testability (extracts pure functions); prevents flake systemically. |
| **Interview-ready** | Given a function, picks the test layer and names the trap (time/random/env dispatch). |

Gap-filler: [`../05-quality-engineering/01-testing-strategy.md`](../05-quality-engineering/01-testing-strategy.md).

## Skill: Collaboration & communication

| Level | Behavior |
| --- | --- |
| Junior | Asks "what do I do?"; writes terse PRs. |
| Mid | Asks with hypothesis + evidence; writes PRs with what/why/tested/risks; reviews with graded, kind feedback. |
| Senior | Writes RFCs with rejected alternatives and rollback plans; disagrees with evidence then commits; teaches via review. |
| **Interview-ready** | Tells a 2-minute STAR story ending on impact; reviews a diff grading severity and phrasing kindly. |

Gap-filler: [`../07-career-and-collaboration/`](../07-career-and-collaboration/README.md).

---

## The overall ladder (memorize this)

- **Junior:** *"How do I make it work?"* — locates, explains, makes small changes.
- **Mid:** *"Is this the right pattern, and how does it fail?"* — modifies safely across layers, tests, debugs, reviews.
- **Senior:** *"What does this commit us to, who pays the cost, and how do I shrink the blast radius?"* — critiques, designs migrations, reduces risk, teaches.

Mid-level interviews test whether you've reached the second question and can gesture at the third. Every rubric above is a concrete way to check yourself against that bar — using a real system you can point at.

## Self-assessment checklist (print this)

- [ ] I can trace Umami's collection and metrics-read flows without notes.
- [ ] I can name 8+ patterns with a tradeoff and failure mode each.
- [ ] I can explain cookieless identity, dual persistence, and the IDOR guard with anchors.
- [ ] I can run a debugging round narrating, splitting the space first.
- [ ] I can review a diff, grade severity, and phrase feedback kindly.
- [ ] I have 5 concrete STAR stories under 2 minutes each.
- [ ] For any concept, I reach for a Umami example + tradeoff + failure mode.

Seven checks = interview-ready for a mid-level fullstack role.
