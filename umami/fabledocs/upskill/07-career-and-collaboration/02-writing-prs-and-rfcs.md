# Writing PRs and RFCs

Clear written communication about code is a mid-level differentiator. This page gives you Umami-tailored templates and the judgment of *when* to write which.

## First: this repo's actual contribution rules

From [`CONTRIBUTING.md`](../../../CONTRIBUTING.md):
- **Target the `dev` branch, never `master`.** PRs against `master` get retargeted.
- Branch from `dev`: `git checkout dev && git pull && git checkout -b my-feature`.
- **One logical change per PR** — keep it focused.
- Must build and lint clean: `pnpm install && pnpm build && pnpm lint`.
- Bug reports need repro steps, expected vs. actual, and environment (version, DB, browser).
- Feature requests: describe the *use case* before the solution.

Read those before your first PR — following the project's stated process is the cheapest way to look competent.

## The PR description template

A good PR answers four questions. Umami's `CONTRIBUTING.md` asks for "what changed and why"; go a little further:

```markdown
## What
One or two sentences: what this PR does.

## Why
The problem or motivation. Link the issue. What's the user/operator impact?

## How
Key implementation decisions a reviewer should know. Call out anything
non-obvious (a workaround, a chosen tradeoff, a pattern you followed).

## Testing
How you verified it. Commands run, cases covered, deny/failure path tested.
(e.g. "pnpm test src/lib/detect.test.ts; added CIDR-match case; manually
verified 401 for cross-tenant request.")

## Risks / follow-ups
Blast radius, what could break, what you deliberately left out, and any
follow-up work. Note if it touches a public contract or both SQL branches.
```

Why this shape: reviewers approve faster when they don't have to reverse-engineer intent. The "Risks" section signals seniority — it shows you thought about blast radius, not just the diff.

## Good vs. weak PR descriptions

**Weak:** *"Fixes the sessions query."* (What was wrong? How? Tested how? Risky?)

**Strong:**
> **What:** Bounds `pageSize` to 1000 on paginated endpoints.
> **Why:** `pageSize` had no upper bound (schema.ts:79), so a client could request an enormous page and pin the DB. No legit caller needs >1000 (grepped usages).
> **How:** Added `.max(1000)` to `pagingParams`. Followed the existing Zod-bound pattern.
> **Testing:** `pnpm test`; added a case asserting `pageSize=5000` is rejected.
> **Risks:** Small. If some internal caller needs a bigger page we'd 400 it — I checked and none does. Rollback = revert one line.

## Commit messages

- **Subject:** imperative, ≤~50 chars: "Bound pageSize on paginated endpoints." Not "fixed stuff."
- **Body (when needed):** why, not what (the diff shows what). Wrap ~72 cols.
- **One logical change per commit** where practical — mirrors the one-change-per-PR rule and makes review and revert clean.

## When to write an RFC instead of just a PR

Write an RFC (a design proposal reviewed *before* code) when the change is:
- **Architectural** — changes a boundary, a data model, or a core abstraction (e.g. the outbox, schema-per-tenant, replacing EAV).
- **Cross-cutting** — touches many files or teams' assumptions.
- **Irreversible-ish** — a migration that's hard to roll back.
- **Contested** — reasonable people would disagree on the approach.

A one-line fix never needs an RFC. "Store event data as JSON instead of EAV" absolutely does ([Review Kata 8](../04-code-reading-gym/04-review-katas.md)) — proposing that as a surprise PR is a classic junior mistake.

## RFC template (tailored to Umami)

```markdown
# RFC: <title>

## Problem
What's wrong or missing today, with anchors (file:line) to the current code.

## Goals / Non-goals
What this will and explicitly will NOT do.

## Proposal
The design. Diagrams if it helps. Which layers change
(schema / query / route / UI / tracker) and how.

## Dual-store impact
Does this touch persistence? How do BOTH the Postgres and ClickHouse
paths change? (Umami-specific: a change that forgets one is a bug.)

## Public-contract impact
Does it affect /api/send, the tracker API, or column widths deployed
trackers depend on? How is it kept backward compatible?

## Alternatives considered
At least one, with why it was rejected. (An RFC with no rejected
alternative reads as unconsidered.)

## Migration & rollback
Additive vs. destructive; expand→migrate→contract; how to roll back.

## Testing strategy
Unit / component / e2e; the deny and failure paths.

## Risks & open questions
Blast radius, unknowns, what needs a decision.
```

The Umami-specific sections (dual-store, public-contract) are what a generic RFC template misses and what a maintainer of *this* repo cares about most.

## Drill

1. Take one good-first-ticket you'd actually do (say Ticket 5, bounding `pageSize`) and write its full PR description using the template.
2. Take one senior project (S1, the outbox) and write its RFC using the template — including a rejected alternative and the dual-store/public-contract sections. This is a reusable interview artifact.

## Interview angle

- "How do you write a good PR?" → the four questions + the risks section signaling blast-radius awareness.
- "When would you write a design doc vs. just code?" → the RFC triggers above.
- "Walk me through a technical proposal you made." → present an RFC you wrote for a kata/project.
