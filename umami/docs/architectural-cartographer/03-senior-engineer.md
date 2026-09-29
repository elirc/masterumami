# Senior Engineer Guide

## Architectural Critique

Scores are 1 to 5, where 5 means strong.

- Scalability: 4. ClickHouse support and hourly rollups improve analytics scale (`src/lib/db.ts:22-36`, `src/queries/sql/pageviews/getPageviewStats.ts:59-99`), but every analytics query must maintain dual implementations (`src/queries/sql/pageviews/getPageviewStats.ts:9-102`).
- TypeScript discipline: 3. Shared query/auth/filter types exist (`src/lib/types.ts:15-120`), but components use `any` in key places such as `WebsiteChart` response destructuring (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:21-21`) and `WebsiteAddForm` submit data (`src/app/(main)/websites/WebsiteAddForm.tsx:17-17`).
- Separation of concerns: 4. Routes delegate request parsing, permissions, and queries (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-33`), but query modules contain substantial SQL string logic (`src/queries/sql/pageviews/getPageviewStats.ts:16-102`).
- Testability: 3. Vitest is configured for `src/**/*.test.{ts,tsx}` with jsdom setup (`vitest.config.ts:10-14`), and Cypress scripts exist (`package.json:44-45`), but critical route/query behavior is not obviously unit-tested in inspected anchors.
- Maintainability: 4. Patterns repeat cleanly across hooks, routes, permissions, and queries (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`).
- Security posture: 3. Auth/permission gates are clear (`src/lib/auth.ts:17-60`, `src/permissions/website.ts:7-40`), but raw SQL construction needs careful review even with placeholder helpers (`src/lib/prisma.ts:255-282`, `src/lib/clickhouse.ts:265-287`).
- Performance: 4. Query parallelism and ClickHouse rollups are good (`src/app/api/websites/[websiteId]/pageviews/route.ts:30-33`, `src/queries/sql/pageviews/getPageviewStats.ts:80-99`), but memo dependencies and division-by-zero rendering need cleanup (`src/components/hooks/useDateRange.ts:13-24`, `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`).

## Performance Audit

Finding 1: `useDateRange` depends on the entire `options` object, so inline callers like `useDateRange({ timezone: timezone })` can force recalculation on every render (`src/components/hooks/useDateRange.ts:8-24`, `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-16`).

Corrected snippet:

```ts
export function useDateRange(options: { ignoreOffset?: boolean; timezone?: string } = {}) {
  const { ignoreOffset = false, timezone } = options;
  const dateRange = useMemo(() => {
    const dateRangeObject = parseDateRange(date || getItem(DATE_RANGE_CONFIG) || DEFAULT_DATE_RANGE_VALUE, unit, locale, timezone);
    return !ignoreOffset && offset ? getOffsetDateRange(dateRangeObject, +offset) : dateRangeObject;
  }, [date, unit, offset, locale, timezone, ignoreOffset]);
}
```

Finding 2: `WebsiteMetricsBar` divides by `visits` and `comparison.visits` without guarding zero (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`).

Corrected snippet:

```ts
const bounceRate = visits ? (Math.min(visits, bounces) / visits) * 100 : 0;
const previousBounceRate = comparison.visits
  ? (Math.min(comparison.visits, comparison.bounces) / comparison.visits) * 100
  : 0;
const duration = visits ? totaltime / visits : 0;
const previousDuration = comparison.visits ? comparison.totaltime / comparison.visits : 0;
```

## Security Audit

Finding 1: Routes should keep the pattern `parseRequest`, `error`, `permission`, then data access. The pageviews route follows it (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-33`), and the website creation route follows it with body validation and permissions (`src/app/api/websites/route.ts:38-69`).

Corrected route skeleton:

```ts
const { auth, query, error } = await parseRequest(request, schema);
if (error) return error();
const { websiteId } = await params;
if (!(await canViewWebsite(auth, websiteId))) return unauthorized();
return json(await readAuthorizedData(websiteId, query));
```

Finding 2: Client fetch assumes JSON response bodies (`src/lib/fetch.ts:33-41`). Non-JSON errors can mask the real backend failure.

Corrected snippet:

```ts
const text = await res.text();
let data;
try {
  data = text ? JSON.parse(text) : null;
} catch {
  data = { error: { status: res.status, message: text || res.statusText } };
}
return { ok: res.ok, status: res.status, data };
```

Finding 3: Share tokens are only accepted when share context header is present (`src/lib/auth.ts:42-48`). New share routes must preserve this boundary and not treat share tokens like full user sessions.

## TypeScript Discipline Review

Good: domain auth and filters are centralized (`src/lib/types.ts:15-120`). Good: feature hooks hide query key/fetch details from callers (`src/lib/types.ts:1-9`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`). Weak spot: `any` appears at UI boundaries where schema-derived types would help, such as `WebsiteChart` data casting (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:21-21`) and form submit data (`src/app/(main)/websites/WebsiteAddForm.tsx:17-17`).

## Custom Abstractions Inventory

- `useApi`: wraps fetch, auth headers, share headers, base path, React Query, and mutations (`src/components/hooks/useApi.ts:20-74`).
- `parseRequest`: combines query/body parsing, Zod validation, and auth (`src/lib/request.ts:12-52`).
- `getQueryFilters`: turns URL params into analytics query filters, including segments and cohorts (`src/lib/request.ts:111-178`).
- `runQuery`: chooses ClickHouse, Kafka, or Prisma path based on environment (`src/lib/db.ts:22-36`).
- `prisma.rawQuery`: replaces custom `{{param}}` placeholders and calls Prisma raw SQL (`src/lib/prisma.ts:255-282`).
- `clickhouse.rawQuery`: sends ClickHouse SQL with query params and JSONEachRow output (`src/lib/clickhouse.ts:265-287`).

## Testing Assessment And One Runnable Test

Vitest includes `src/**/*.test.{ts,tsx}` (`vitest.config.ts:10-14`). A risky untested behavior from inspected code is URL construction in hooks/routes, but pure helper tests are easier to run. Add this test if you harden `src/lib/fetch.ts` JSON parsing:

```ts
// src/lib/fetch.test.ts
import { describe, expect, it, vi } from 'vitest';
import { request } from './fetch';

describe('request', () => {
  it('returns parsed JSON response data', async () => {
    vi.stubGlobal('fetch', vi.fn().mockResolvedValue({
      ok: true,
      status: 200,
      json: async () => ({ ok: true }),
    }));

    await expect(request('GET', '/api/heartbeat')).resolves.toEqual({
      ok: true,
      status: 200,
      data: { ok: true },
    });
  });
});
```

## Bug Injection Exercise

Do not modify production code. Reproduce each by writing a test or using the UI.

1. Symptom: chart does not update when a filter changes. Test scenario: change URL filter and assert query key changes. Inspect `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`.
2. Symptom: bounce rate shows `NaN%`. Test scenario: mock stats response with `visits: 0`. Inspect `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`.
3. Symptom: share link works in normal app but fails in `/share`. Test scenario: omit share context header. Inspect `src/lib/auth.ts:42-48` and `src/components/hooks/useApi.ts:25-33`.
4. Symptom: date changes cause excessive chart recomputation. Test scenario: render `WebsiteChart` repeatedly with same timezone. Inspect `src/components/hooks/useDateRange.ts:13-24`.
5. Symptom: non-JSON 500 response crashes client request parsing. Test scenario: mock fetch response where `json()` throws. Inspect `src/lib/fetch.ts:33-41`.

## Git History Learning Exercise

Use these realistic commit messages as reading prompts:

1. `Add ClickHouse hourly pageview rollups`: implies a shift from raw-event scans to aggregate reads; inspect `src/queries/sql/pageviews/getPageviewStats.ts:59-99`.
2. `Introduce share-token context header`: implies public dashboards needed constrained auth; inspect `src/lib/auth.ts:42-48`.
3. `Move analytics filters into shared request helper`: implies route duplication was reduced; inspect `src/lib/request.ts:111-178`.
4. `Add session replay models`: implies schema expansion around replay chunks and saved replays; inspect `prisma/schema.prisma:362-400`.
5. `Use React Query provider defaults`: implies consistency in server-data caching; inspect `src/app/Providers.tsx:11-19`.

## If I Owned This Codebase

- Guard metrics division by zero. Effort: S. Impact: high user-facing polish. Lines: `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`.
- Replace key `any` UI casts with route response types. Effort: M. Impact: medium reliability. Lines: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:21-46`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-13`.
- Harden fetch JSON parsing. Effort: S. Impact: medium debugging clarity. Lines: `src/lib/fetch.ts:18-41`.
- Add route-level tests for permission and schema failures. Effort: M. Impact: high security confidence. Lines: `src/app/api/websites/[websiteId]/pageviews/route.ts:16-28`.
- Document filter-to-SQL extension process next to `src/lib/request.ts` and query helpers. Effort: S. Impact: medium onboarding speed. Lines: `src/lib/request.ts:79-178`, `src/lib/prisma.ts:74-150`, `src/lib/clickhouse.ts:73-140`.
- Track dual-query parity with tests or snapshots. Effort: L. Impact: high analytics correctness. Lines: `src/queries/sql/pageviews/getPageviewStats.ts:16-102`.

