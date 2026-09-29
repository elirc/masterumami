# Testing Strategy

## The test layers here

Umami uses two layers, deliberately:

| Layer | Tool | Runs | Tests | Where |
| --- | --- | --- | --- | --- |
| **Unit / component** | Vitest (+ Testing Library, jsdom) | fast, no server/DB | pure functions, React components | `*.test.ts(x)` beside source |
| **End-to-end** | Cypress | against a running app + DB | real user/API flows | `cypress/e2e/*.cy.ts` |

There is **no dedicated integration layer** (no in-process API-route tests with a test DB). API behavior is covered at the e2e level (`cypress/e2e/api-*.cy.ts`). This is a deliberate shape for a small team, and a gap you can fill (tickets exist for isolated route/permission tests).

## What belongs at each layer

**Unit** — the majority. Anything pure or logic-heavy:
- Formatting/parsing (`format.test.ts`, `detect.test.ts`, `get-base-url.test.ts`, `match-configured-path.test.ts`).
- Pure business rules (`boards.test.ts` — the component-compatibility matrix).
- Data transforms (`flattenJSON`, `getStringValue` — currently untested, Ticket 7).
- Guards (`hasBlockedIp`, `parseToken` fail-closed — Tickets 1, 20).

**Component** — React pieces with the custom render:
- Forms, tables, interactive components using the providers in `src/test/render.tsx` (which wraps React Query, next-intl, the Zen design system, router).

**E2E** — cross-cutting user journeys and API contracts:
- Login/logout, website/user/team CRUD (`cypress/e2e/*`).
- Cross-tenant authorization (a good place to add IDOR regression tests).

## What NOT to test

- **Third-party libraries** (Prisma, Zod, React Query) — test *your usage*, not their internals.
- **The database itself** — don't assert Postgres does joins.
- **Generated code** (`src/generated/prisma`).
- **Trivial getters/constants** — no behavior to protect.
- **Exact SQL strings** — brittle; test the *result* of a query against seeded data (e2e), or the *inputs* to the query builder (unit), not the string.

The rule: test behavior and contracts, not implementation details. A test that breaks when you rename a private variable is a bad test.

## Fixtures, builders, mocks, isolation

Umami's test infrastructure ([`src/test/`](../../src/test/)):

- **`setup.ts`** — global test setup: configures `data-test` as the test id, mocks browser APIs jsdom lacks (`matchMedia`, `ResizeObserver`, `IntersectionObserver`, `clipboard`, `scrollTo`), and `afterEach` does `cleanup()` + `vi.restoreAllMocks()` for **isolation** (each test starts clean).
- **`render.tsx`** — a custom `render` wrapping components in all required providers (React Query, intl, Zen, router), plus `createTestQueryClient()` with `retry: false` so tests don't wait on retries. Always use this, not Testing Library's raw `render`.
- **`navigation.ts`** — test navigation control (`setTestUrl`, `resetTestNavigation`) so components depending on the route work.

**Mocking:** `vi.fn()`/`vi.mock()` from Vitest. For write-path unit tests you'd mock `@/lib/prisma` to avoid a real DB (Ticket 2).

**Isolation:** the `afterEach` cleanup + `restoreAllMocks` is what keeps tests independent — no shared mutable state leaking between them. When you add tests, rely on this rather than manual teardown.

## Time & randomness (flake sources)

Two things make tests flaky; both appear in Umami's logic:

- **Time:** identity derivation and visit expiry depend on `Date.now()`/`createdAt` ([`send/route.ts:140-175`](../../src/app/api/send/route.ts)). Test time-dependent logic with `vi.setSystemTime()` / fake timers, never real `Date.now()`.
- **Randomness:** the recorder samples with `Math.random()` ([`recorder/index.js:22`](../../src/recorder/index.js)); `uuid()` v4 is random ([`crypto.ts:65`](../../src/lib/crypto.ts)). Stub or seed randomness (`vi.spyOn(Math,'random')`) so assertions are deterministic. The deterministic `uuid(...args)` path (v5 hash) is naturally testable — prefer asserting on it.

## Flake prevention checklist

- [ ] No real `Date.now()`/`Math.random()` in assertions — fake them.
- [ ] Use `createTestQueryClient()` (retry off) for component tests.
- [ ] Rely on `afterEach` cleanup; don't share state across tests.
- [ ] `waitFor`/`findBy*` for async UI, never fixed `setTimeout`.
- [ ] E2E: use `data-test` selectors (stable), not text/CSS that changes.
- [ ] Reset env vars you set (`process.env.X`) within the test.

## The testing pyramid, Umami-shaped

```
        /\        e2e (Cypress)      few, slow, high-confidence  — cypress/e2e
       /  \
      /----\      component (Vitest+RTL)  some — *.test.tsx with render.tsx
     /      \
    /--------\    unit (Vitest)      many, fast, cheap — *.test.ts
```

The pyramid is intentionally bottom-heavy: most confidence should come from fast unit tests, with e2e reserved for the flows that only make sense end to end (login, cross-tenant access).

## Drill

Categorize these five hypothetical tests by layer and say which are worth writing: (a) "`getSalt('month')` is stable within a month," (b) "Prisma inserts a row," (c) "login form shows an error on wrong password," (d) "a user gets 401 for another's website," (e) "the exact SQL string of `getWebsiteStats`." Answers: a=unit(yes), b=don't(third-party), c=component or e2e(yes), d=e2e(yes, high value), e=don't(brittle).

## Interview angle

- "How do you decide what to test at which layer?" → the pyramid + "test behavior not implementation."
- "How do you handle time/randomness in tests?" → fake timers, seed random, prefer deterministic paths.
- "What wouldn't you test?" → third-party, generated, brittle SQL strings.
See [`02-writing-tests-here.md`](02-writing-tests-here.md) to actually write them.
