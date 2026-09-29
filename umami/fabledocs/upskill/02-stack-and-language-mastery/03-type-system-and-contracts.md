# Type System & Contracts

## The core idea: types are contracts, and the edge is runtime

TypeScript is compile-time only — it erases at runtime. So types guarantee nothing about data that *enters* your program (an HTTP body, a DB row, an env var). The senior mental model: **validate at the boundary, trust types inside.** Umami does exactly this — Zod schemas validate incoming data and produce typed objects; internal code then relies on those types.

## Zod as the runtime edge of the type system

A Zod schema is a value that both *validates* at runtime and *infers* a static type. Umami's `src/lib/schema.ts` is the boundary contract for the whole API.

- **Parse, don't validate-and-hope:** `schema.safeParse(input)` returns either typed `data` or an `error` ([`request.ts:26-41`](../../src/lib/request.ts)). Downstream code uses `result.data`, which is typed — the untyped `input` never leaks in.
- **Coercion at the edge:** `z.coerce.number()`, `z.coerce.date()` turn query strings into typed values ([`schema.ts:17-19`](../../src/lib/schema.ts)) — because query params are always strings.
- **Cross-field invariants:** `.refine`/`.superRefine` encode rules a single field can't (exactly-one-source, [`send/route.ts:53-63`](../../src/app/api/send/route.ts); date-range presence, [`schema.ts:32-42`](../../src/lib/schema.ts)).
- **Discriminated unions:** `z.discriminatedUnion('type', [...])` ([`schema.ts:279-289`](../../src/lib/schema.ts)) validates a polymorphic payload precisely and gives you a narrowed static type per variant.

The transferable skill: whenever data crosses into your system, there should be a schema. "Where's your validation boundary?" is a real interview question, and Umami's answer is `parseRequest` + `schema.ts`.

## Inference, narrowing, and unions

- **Inference:** TS infers types so you rarely annotate locals. Umami annotates *boundaries* (function params, exported types like `WebsiteStatsData`, `saveEvent.ts:9-15`) and lets inference handle the rest.
- **Narrowing:** control-flow narrows a union. In `checkAuth` ([`auth.ts:23-33`](../../src/lib/auth.ts)), `const { userId, authKey } = payload || {}` then branching on which is present narrows the path. The discriminant `share.shareType` in `share/[slug]/route.ts:64-92` narrows what data to gather.
- **Unions & literals:** `ROLES`/`PERMISSIONS` are `as const` objects ([`constants.ts:164-184`](../../src/lib/constants.ts)), so their values are literal-typed strings, not just `string` — enabling exhaustive checks and autocomplete.

## `as const`, `satisfies`, `unknown` vs `any`

- **`as const`** freezes an object into readonly literal types — used pervasively for constants (`EVENT_TYPE`, `OPERATORS`, `ROLE_PERMISSIONS`). This is what lets `EVENT_TYPE.pageView` be the literal `1`, not `number`.
- **`satisfies`** (a TS 4.9+ tool) checks a value against a type *without widening it* — you keep the precise inferred type while proving it conforms. Look for opportunities: many `as` casts in the query layer could be `satisfies` for more safety. This is a great "how would you improve type safety here?" answer.
- **`unknown` vs `any`:** `any` disables checking (contagious, dangerous); `unknown` forces you to narrow before use (safe). Umami has pragmatic `any` in the query engine (`rawQuery(sql: string, data: Record<string, any>)`, [`prisma.ts:255`](../../src/lib/prisma.ts)) and in `parseRequest`'s return (`Promise<any>`, [`request.ts:16`](../../src/lib/request.ts)) — honest tradeoffs where dynamic SQL/data makes precise typing costly. Recognizing *where `any` is a reasonable escape hatch vs. a smell* is a mid-level judgment.

## Generated types & the DB contract

Prisma generates a fully typed client from `schema.prisma` into `src/generated/prisma`. Query files import model types (`import type { Website } from '@/generated/prisma/client'`, [`load.ts:1`](../../src/lib/load.ts)) and input types (`Prisma.SessionCreateInput`, [`createSession.ts:6`](../../src/queries/sql/sessions/createSession.ts)). This is a second contract: the schema is the single source of truth, and the generated types propagate it through the app. Changing a column ripples into type errors at every use — a feature, not a bug.

Note the seam: the *raw* SQL layer (`rawQuery`) returns `any`, so results of hand-written analytics queries are **not** type-checked against the schema. That's the cost of raw SQL — you trade type safety for expressiveness (see [Pattern 2](../03-architecture-and-patterns/05-pattern-catalog.md#pattern-2-parameterized-raw-sql-template)). Typing those return shapes (as the query files do with explicit interfaces like `WebsiteStatsData`) is the mitigation.

## Pitfall checklist (TS)

- [ ] Trusting a type on data that crossed a boundary without a schema.
- [ ] `any` spreading through a call chain (should be `unknown` + narrowing).
- [ ] Casting with `as` where `satisfies` would keep safety.
- [ ] Forgetting query params are strings (need `z.coerce`).
- [ ] Assuming raw-SQL results match the schema types (they're `any`).
- [ ] A discriminated union validated in one place but bypassed by `anyObjectParam` in another (`schema.ts:276` vs `:279`).

## Drills

1. Trace one API value from HTTP string to typed use: pick `startAt`. Where is it coerced (`schema.ts:17`), where does it become a `Date` (`request.ts:66`), where is it used? Name the boundary that makes the internal type trustworthy.
2. Find three `any`s in `src/lib/`. For each, decide: reasonable escape hatch or should-be-`unknown`? Justify.
3. Pick a query file with an explicit result interface (e.g. `getWebsiteStats.ts:9-15`). What guarantees that interface matches the SQL's actual columns? (Answer: nothing but discipline — that's the raw-SQL cost.)

## Interview angle

Prepares you for (see [`../08-interview-prep/01-js-ts-node-deep-dive.md`](../08-interview-prep/01-js-ts-node-deep-dive.md)):
- "Where's your validation boundary, and why validate if you have types?" → Zod + `parseRequest`.
- "`unknown` vs `any`; when is `any` acceptable?" → the raw-SQL layer.
- "How do discriminated unions help?" → Q12, `reportTypeSchema`.
- "How do you keep DB and app types in sync?" → Prisma generation, and the raw-SQL gap.
