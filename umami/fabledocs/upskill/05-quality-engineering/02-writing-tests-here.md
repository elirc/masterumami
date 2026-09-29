# Writing Tests Here

Eight recipes using Umami's actual tooling. Each shows the shape, the command, and the trap. Snippets marked illustrative are patterns to adapt, following the real style in `src/lib/*.test.ts` and `src/test/render.tsx`.

Commands: `pnpm test` (all), `pnpm test <path>` (one file), `pnpm test:watch`. __inferred__ (no deps installed in this clone). Cypress: `pnpm cypress-open`.

---

## Recipe 1 — Pure function (happy path)

The bread-and-butter unit test. Follow `format.test.ts` / `boards.test.ts` exactly.

```ts
// Illustrative fake code: not from this repo — mirrors src/lib/detect.test.ts style
import { expect, test } from 'vitest';
import { getDevice } from './detect';

test('classifies a narrow desktop as laptop', () => {
  const ua = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...';
  expect(getDevice(ua, '1440x900')).toBe('laptop'); // ≤1920 width → laptop
});
```

**Command:** `pnpm test src/lib/detect.test.ts`
**Trap:** use realistic UA strings — `ua-parser-js` returns different device types for junk input.
**Real anchor:** `getDevice` logic at [`detect.ts:49-61`](../../src/lib/detect.ts).

---

## Recipe 2 — Validation failure (Zod)

Assert bad input is rejected. Tests the boundary contract directly.

```ts
// Illustrative fake code: not from this repo
import { expect, test } from 'vitest';
import { z } from 'zod';
import { pagingParams } from './schema';

test('rejects non-positive pageSize', () => {
  const schema = z.object(pagingParams);
  expect(schema.safeParse({ pageSize: '-5' }).success).toBe(false);
  expect(schema.safeParse({ pageSize: '10' }).success).toBe(true); // coerced from string
});
```

**Command:** `pnpm test src/lib/schema.test.ts`
**Trap:** query params are strings — test `'10'` not `10` to exercise `z.coerce` ([`schema.ts:78`](../../src/lib/schema.ts)).
**Ties to:** Ticket 5 (adding a `pageSize` max).

---

## Recipe 3 — A write with the DB mocked (permission/truncation)

Unit-test write logic without a database by mocking `@/lib/prisma`.

```ts
// Illustrative fake code: not from this repo
import { expect, test, vi, beforeEach } from 'vitest';

vi.mock('@/lib/prisma', () => ({
  default: { client: { websiteEvent: { create: vi.fn() } } },
}));

beforeEach(() => { delete process.env.CLICKHOUSE_URL; }); // force the Prisma branch

test('truncates event name to 50 chars', async () => {
  const prisma = (await import('@/lib/prisma')).default;
  const { saveEvent } = await import('./saveEvent');
  await saveEvent({ /* ...args..., */ eventName: 'x'.repeat(60) } as any);
  const arg = (prisma.client.websiteEvent.create as any).mock.calls[0][0];
  expect(arg.data.eventName).toHaveLength(50);
});
```

**Command:** `pnpm test src/queries/sql/events/saveEvent.test.ts`
**Trap:** `runQuery` dispatches by env — clear `CLICKHOUSE_URL` and set a Postgres `DATABASE_URL` so the relational branch runs ([`db.ts:22-36`](../../src/lib/db.ts)).
**Ties to:** Ticket 2. **Real anchor:** truncation at [`saveEvent.ts:131`](../../src/queries/sql/events/saveEvent.ts).

---

## Recipe 4 — Permission-failure logic

Test that `canViewWebsite` denies a non-owner (mock the entity/team reads).

```ts
// Illustrative fake code: not from this repo
import { expect, test, vi } from 'vitest';

vi.mock('@/queries/prisma', () => ({
  getEntity: vi.fn(async () => ({ userId: 'owner-1' })),
  getTeamUser: vi.fn(async () => null),
}));

test('denies a user who is not the owner', async () => {
  const { canViewWebsite } = await import('./website');
  const ok = await canViewWebsite({ user: { id: 'someone-else' } } as any, 'w1');
  expect(ok).toBe(false);
});
```

**Command:** `pnpm test src/permissions/website.test.ts`
**Trap:** `getEntity` is imported from `@/lib/entity` in the real code — mock the exact module path used ([`website.ts:3`](../../src/permissions/website.ts)).
**Value:** this is the IDOR guard; a unit test here is high-leverage.

---

## Recipe 5 — A component with the custom render (UI state)

Use `src/test/render.tsx`, never Testing Library's raw `render` — components need the providers.

```tsx
// Illustrative fake code: not from this repo — uses the repo's custom render
import { expect, test } from 'vitest';
import { render, screen } from '@/test/render'; // the wrapped render
import { SomeForm } from './SomeForm';

test('shows a required error on empty submit', async () => {
  const { user } = render(<SomeForm />);
  await user.click(screen.getByTestId('button-submit'));
  expect(await screen.findByText(/required/i)).toBeVisible();
});
```

**Command:** `pnpm test src/app/.../SomeForm.test.tsx`
**Trap:** the test id attribute is `data-test` (configured in [`setup.ts`](../../src/test/setup.ts)); `getByTestId('button-submit')` maps to `data-test="button-submit"`. Use `findBy*` for async.
**Real anchor:** provider wrapping in [`render.tsx`](../../src/test/render.tsx).

---

## Recipe 6 — Cross-tenant rejection (e2e)

The highest-value security test. Follow `cypress/e2e/api-website.cy.ts`.

```ts
// Illustrative fake code: not from this repo — Cypress e2e shape
describe('cross-tenant isolation', () => {
  it('denies reading another user\'s website', () => {
    cy.loginAs('userA');
    cy.request({ url: '/api/websites/<userB-website-id>/stats?...', failOnStatusCode: false })
      .its('status').should('eq', 401);
  });
});
```

**Command:** `pnpm cypress-run` (needs a running app + seeded users).
**Trap:** `failOnStatusCode: false` so Cypress doesn't abort on the expected 401.
**Value:** pins the IDOR guard against regressions across every future route change.

---

## Recipe 7 — Async side effect (fire-and-forget)

Assert the chosen behavior when a downstream send fails.

```ts
// Illustrative fake code: not from this repo
import { expect, test, vi } from 'vitest';

test('kafka send failure does not throw', async () => {
  vi.mock('kafkajs'); // producer.send rejects
  const kafka = (await import('@/lib/kafka')).default;
  await expect(kafka.sendMessage('event', { a: 1 })).resolves.toBeUndefined();
});
```

**Command:** `pnpm test src/lib/kafka.test.ts`
**Trap:** this pins the *current* (swallow-and-return-undefined) behavior. If you implement the outbox fix, this test must change to assert the new guarantee — which is exactly the point of the test.
**Real anchor:** [`kafka.ts:87-90`](../../src/lib/kafka.ts).

---

## Recipe 8 — Time-dependent logic (visit expiry / salt)

Freeze time so identity/expiry assertions are deterministic.

```ts
// Illustrative fake code: not from this repo
import { expect, test, vi, afterEach } from 'vitest';
import { getSalt } from './crypto';

afterEach(() => vi.useRealTimers());

test('salt is stable within a month, changes across it', () => {
  const jan = new Date('2026-01-15T00:00:00Z');
  const feb = new Date('2026-02-15T00:00:00Z');
  expect(getSalt('month', jan)).toBe(getSalt('month', jan));
  expect(getSalt('month', jan)).not.toBe(getSalt('month', feb));
});
```

**Command:** `pnpm test src/lib/crypto.test.ts`
**Trap:** `getSalt` takes the date explicitly (good — no clock mocking needed here). For the *visit-expiry* logic in `send/route.ts`, you'd need `vi.setSystemTime()` because it reads `Date.now()` ([`send/route.ts:141`](../../src/app/api/send/route.ts)).
**Ties to:** Tickets 15, 17.

---

## Running the suite

```bash
pnpm test                         # all unit/component  __inferred__
pnpm test src/lib/format.test.ts  # one file
pnpm test:watch                   # watch mode
pnpm cypress-open                 # interactive e2e
pnpm lint && pnpm check           # Biome lint + format
```

## Drill

Take Ticket 1 (test `hasBlockedIp`) and Ticket 12 (test `getDevice`) and actually write them following Recipes 1–2. Run `pnpm test`. If you can't run it (no deps), hand-trace your assertions against the source and predict pass/fail per case. Then write Recipe 4 (the permission test) — it's the one that most resembles a real interview take-home.

## Interview angle

- "How would you test this function?" → pick the recipe, name the trap (time/random/env dispatch).
- "How do you test a component that needs context?" → the custom render + providers.
- "How do you test authorization?" → unit the predicate (Recipe 4) *and* e2e the cross-tenant 401 (Recipe 6).
