# Tier 3: Senior Missions

### Mission 18: Reverse-Engineer the Architecture Decisions
**Tier:** Senior
**Time Estimate:** 70 minutes
**Goal:** Infer why the system is shaped this way.
**The Concept:** Architecture is the pattern of bets. Umami bets on Next App Router, query hooks, domain permissions, and dual analytics storage.
**Design Intent Before You Read the Code:** Read provider setup, DB dispatch, query implementation, and schema indexes. Ask what each decision optimizes.
**Find It In The Code:** Exact files and ranges: `src/app/Providers.tsx:11-62`, `src/lib/db.ts:22-36`, `src/queries/sql/pageviews/getPageviewStats.ts:59-99`, `prisma/schema.prisma:135-149`.

```ts
// src/lib/db.ts:22-36
export async function runQuery(queries: any) {
  if (process.env.CLICKHOUSE_URL) return queries[CLICKHOUSE](); // Prefer analytics store when configured.
  const db = getDatabaseType();
  if (db === POSTGRESQL) return queries[PRISMA](); // Default install path.
}
```
**The Aha Moment:** The system optimizes for simple PostgreSQL setup and scalable ClickHouse analytics.
**Socratic Checkpoint:** 1. Why support two DB paths? 2. Why put React Query globally? 3. Why index `websiteId` plus `createdAt`? 4. What does App Router buy here? 5. What complexity does this create?

How to self-grade: Strong answers cite storage dispatch, query duality, provider defaults, and schema indexes.
**Connects To:** Mission 19 and Mission 24, because architecture decisions predict bugs and history.

### Mission 19: Find the Bugs Before They Happen
**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Predict likely bugs from code shape.
**The Concept:** Senior debugging is weather forecasting: you read pressure changes before the storm hits users.
**Design Intent Before You Read the Code:** Inspect metrics math, memo dependencies, fetch parsing, and auth boundaries.
**Find It In The Code:** Exact files and ranges: `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`, `src/components/hooks/useDateRange.ts:13-24`, `src/lib/fetch.ts:33-41`, `src/lib/auth.ts:42-48`.

```tsx
// src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61
value: (Math.min(visits, bounces) / visits) * 100, // visits=0 becomes NaN.
value: totaltime / visits, // visits=0 becomes Infinity or NaN.
```

```ts
// src/lib/fetch.ts:33-41
}).then(async res => {
  const data = await res.json(); // Non-JSON responses throw before the wrapper returns.
  return { ok: res.ok, status: res.status, data };
});
```
**The Aha Moment:** Many bugs are visible before you run the app if you read assumptions.
**Socratic Checkpoint:** 1. Where can zero division happen? 2. Where can JSON parsing hide errors? 3. Where can memo dependencies churn? 4. What protects share tokens? 5. Which bug would users notice first?

How to self-grade: Strong answers identify symptom, root line, and a small guard for each issue.
**Connects To:** Mission 20 and Mission 21, because bug prediction leads to test scenarios and performance checks.

### Mission 20: The Bug Injection Challenge
**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Design tests from user-visible symptoms.
**The Concept:** Injecting bugs in thought is a rehearsal for production debugging.
**Design Intent Before You Read the Code:** Do not edit production code. Write symptoms and tests against the risky boundaries.
**Find It In The Code:** Exact files and ranges: `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`, `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`, `src/lib/auth.ts:42-48`, `src/components/hooks/useDateRange.ts:13-24`, `src/lib/fetch.ts:33-41`.

```ts
// Test idea for src/lib/fetch.ts:33-41
// Mock fetch().json() throwing and assert the wrapper surfaces a controlled error.
// This test describes the user symptom: "dashboard fails with an unhelpful parsing crash."
```

```tsx
// Test idea for src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61
// Mock stats with visits=0 and assert no NaN or Infinity text appears.
// This test describes the user symptom: "metrics show broken percentages on empty sites."
```
**The Aha Moment:** A good bug exercise starts with what the user sees, then proves the risky line.
**Socratic Checkpoint:** 1. What is the symptom? 2. What line likely caused it? 3. What input reproduces it? 4. What assertion proves the fix? 5. What should not be mocked?

How to self-grade: Strong answers include a user symptom, minimal test input, and exact source range for each bug.
**Connects To:** Mission 23, because bug injection becomes missing-test work.

### Mission 21: Performance X-Ray
**Tier:** Senior
**Time Estimate:** 65 minutes
**Goal:** Find performance choices and risks.
**The Concept:** Performance in analytics is about not recounting every event when a summary shelf exists.
**Design Intent Before You Read the Code:** Inspect query parallelism, rollups, React memoization, and provider caching.
**Find It In The Code:** Exact files and ranges: `src/app/api/websites/[websiteId]/pageviews/route.ts:30-33`, `src/queries/sql/pageviews/getPageviewStats.ts:59-99`, `src/app/Providers.tsx:11-19`, `src/components/metrics/PageviewsChart.tsx:27-83`.

```ts
// src/app/api/websites/[websiteId]/pageviews/route.ts:30-33
const [pageviews, sessions] = await Promise.all([
  getPageviewStats(websiteId, filters), // Independent query.
  getSessionStats(websiteId, filters), // Independent query.
]);
```

```ts
// src/queries/sql/pageviews/getPageviewStats.ts:80-99
from website_event_stats_hourly as website_event
// Uses hourly rollups when filters do not require raw event columns and unit is not minute.
```
**The Aha Moment:** The fastest query is often the one that reads pre-aggregated data.
**Socratic Checkpoint:** 1. Why use `Promise.all`? 2. When does ClickHouse read raw events? 3. When does it read hourly stats? 4. What stale time is configured? 5. What memo dependency looks suspicious?

How to self-grade: Strong answers cite route parallelism, rollup branch, QueryClient defaults, and chart memoization.
**Connects To:** Mission 18 and Mission 23, because performance choices deserve tests or benchmarks.

### Mission 22: The Security Audit
**Tier:** Senior
**Time Estimate:** 75 minutes
**Goal:** Audit a route and auth path for data exposure risks.
**The Concept:** Security is making sure the dashboard only opens the right analytics binder for the right person.
**Design Intent Before You Read the Code:** Inspect token parsing, share context, route validation, website permission, and raw SQL parameterization.
**Find It In The Code:** Exact files and ranges: `src/lib/auth.ts:17-60`, `src/permissions/website.ts:7-40`, `src/app/api/websites/[websiteId]/pageviews/route.ts:16-28`, `src/lib/prisma.ts:255-282`, `src/lib/clickhouse.ts:265-287`.

```ts
// src/lib/auth.ts:42-48
if (!user?.id && shareToken) {
  const shareContext = request.headers.get(SHARE_CONTEXT_HEADER);
  if (!shareContext) return null; // Share token cannot be used as general API auth.
}
```

```ts
// src/lib/prisma.ts:268-276
const query = sql?.replaceAll(/\{\{\s*(\w+)(::\w+)?\s*}}/g, (...args) => {
  const [, name, type] = args;
  params.push(data[name]); // Values become positional parameters.
  return `$${params.length}${type ?? ''}`;
});
```
**The Aha Moment:** Security is a chain; one skipped link can leak analytics.
**Socratic Checkpoint:** 1. What is authenticated? 2. What is authorized? 3. Why require share context? 4. How are SQL values parameterized? 5. What dynamic SQL remains risky?

How to self-grade: Strong answers separate identity, permission, and SQL injection risk.
**Connects To:** Mission 15 and Mission 19, because review and bug prediction protect this chain.

### Mission 23: Write the Test That Doesn't Exist
**Tier:** Senior
**Time Estimate:** 90 minutes
**Goal:** Specify one valuable test for risky behavior.
**The Concept:** A test is a tripwire across a path users care about.
**Design Intent Before You Read the Code:** Choose a risk with stable inputs and observable output. Vitest is configured for `src/**/*.test.{ts,tsx}` with jsdom (`vitest.config.ts:10-14`).
**Find It In The Code:** Exact files and ranges: `vitest.config.ts:10-14`, `src/lib/fetch.ts:18-41`, `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`, `src/lib/request.ts:12-52`.

```ts
// Candidate test for src/lib/fetch.ts:18-41
import { describe, expect, it, vi } from 'vitest';
import { request } from './fetch';

it('wraps a successful JSON response', async () => {
  vi.stubGlobal('fetch', vi.fn().mockResolvedValue({ ok: true, status: 200, json: async () => ({ ok: true }) }));
  await expect(request('GET', '/api/heartbeat')).resolves.toMatchObject({ ok: true, status: 200, data: { ok: true } });
});
```
**The Aha Moment:** The best first test locks down behavior at a boundary.
**Socratic Checkpoint:** 1. What risk are you testing? 2. What is the smallest input? 3. What output proves correctness? 4. Does the test depend on a database? 5. Where would you put the file?

How to self-grade: Strong answers choose a boundary, avoid heavy mocks when possible, and cite config plus source lines.
**Connects To:** Mission 20 and Mission 24, because tests capture the lessons from bugs and evolution.

### Mission 24: The Git History Tells a Story
**Tier:** Senior
**Time Estimate:** 60 minutes
**Goal:** Practice inferring evolution from commit messages and current code.
**The Concept:** History is the product's fossil record: schema, query paths, and scripts show what problems arrived over time.
**Design Intent Before You Read the Code:** Use current code as evidence for plausible commits.
**Find It In The Code:** Exact files and ranges: `src/lib/db.ts:22-36`, `src/queries/sql/pageviews/getPageviewStats.ts:59-99`, `src/lib/auth.ts:42-48`, `scripts/seed-data.ts:4-17`, `prisma/schema.prisma:362-400`.

```text
Commit: Add ClickHouse support for pageview rollups
Evidence: runQuery selects ClickHouse by env, and pageview stats can read website_event_stats_hourly.
Source: src/lib/db.ts:22-36, src/queries/sql/pageviews/getPageviewStats.ts:80-99
```

```text
Commit: Add local seed data generator
Evidence: seed script generates Demo Blog and Demo SaaS and blocks production execution.
Source: scripts/seed-data.ts:4-17, scripts/seed-data.ts:90-104
```
**The Aha Moment:** Current architecture is a record of past product and scale pressures.
**Socratic Checkpoint:** 1. What problem did ClickHouse solve? 2. What problem did share context solve? 3. What problem did seed data solve? 4. What problem did replay schema solve? 5. Which change likely increased maintenance cost?

How to self-grade: Strong answers tie each guessed commit to current code evidence and tradeoffs.
**Connects To:** Mission 18 and the user story build path, because future features become tomorrow's history.

