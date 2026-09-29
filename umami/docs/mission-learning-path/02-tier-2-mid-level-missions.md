# Tier 2: Mid-Level Missions

### Mission 9: State Has a Home and a Reason
**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Decide which state belongs in URL, React Query, or Zustand.
**The Concept:** Analytics state is like dashboard labels: the selected date belongs in the URL, server metrics belong in cache, and user preferences belong in the app store.
**Design Intent Before You Read the Code:** Open `src/store/app.ts:14-54`, `src/app/Providers.tsx:11-19`, and `src/components/hooks/useDateRange.ts:8-37`. State location should match lifetime and shareability.
**Find It In The Code:** Exact files and ranges: `src/store/app.ts:14-54`, `src/app/Providers.tsx:11-19`, `src/components/hooks/useDateRange.ts:8-37`.

```ts
// src/store/app.ts:14-23
const initialState = {
  locale: getItem(LOCALE_CONFIG) || process.env.defaultLocale || DEFAULT_LOCALE, // Preference.
  theme: getItem(THEME_CONFIG) || DEFAULT_THEME, // Preference.
  timezone: getItem(TIMEZONE_CONFIG) || getTimezone(), // Preference.
  shareToken: null,
  user: null,
  config: null,
};
```

```ts
// src/components/hooks/useDateRange.ts:8-37
const { query: { date = '', unit = '', offset = 0, compare = 'prev' } } = useNavigation();
const dateRange = useMemo(() => {
  const dateRangeObject = parseDateRange(date || getItem(DATE_RANGE_CONFIG) || DEFAULT_DATE_RANGE_VALUE, unit, locale, options.timezone);
  return !options.ignoreOffset && offset ? getOffsetDateRange(dateRangeObject, +offset) : dateRangeObject;
}, [date, unit, offset, options]);
return { date, unit, offset, compare, dateRange, dateCompare };
```
**The Aha Moment:** State placement communicates who needs to observe or share that state.
**Socratic Checkpoint:** 1. Which state lives in Zustand? 2. Which state lives in URL query params? 3. Which state lives in React Query? 4. Why is timezone a preference? 5. Why is pageviews data not in Zustand?

How to self-grade: Strong answers cite all three source ranges and explain lifetime, shareability, and server cache invalidation.
**Connects To:** Mission 10 and Mission 14, because hooks turn state into data requests.

### Mission 10: The Custom Hook Ecosystem
**Tier:** Mid-Level
**Time Estimate:** 55 minutes
**Goal:** Understand how app-specific hooks compose.
**The Concept:** Hooks are like analytics instruments: one reads the URL, one reads date settings, one prepares API access, and one asks for a specific report.
**Design Intent Before You Read the Code:** Open `useNavigation`, `useDateRange`, `useApi`, and `useWebsitePageviewsQuery`. Hooks should be small and composable.
**Find It In The Code:** Exact files and ranges: `src/components/hooks/useNavigation.ts:5-58`, `src/components/hooks/useDateRange.ts:8-37`, `src/components/hooks/useApi.ts:20-74`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`.

```ts
// src/components/hooks/queries/useWebsitePageviewsQuery.ts:15-32
const { get, useQuery } = useApi(); // API transport and React Query primitives.
const { startAt, endAt, unit, timezone } = useDateParameters(); // Date query params.
const queryParams = useFilterParameters(); // Analytics filter query params.
return useQuery({
  queryKey: ['websites:pageviews', { websiteId, compare, startAt, endAt, unit, timezone, ...queryParams }],
  queryFn: () => get(`/websites/${websiteId}/pageviews`, { compare, startAt, endAt, unit, timezone, ...queryParams }),
});
```
**The Aha Moment:** A feature hook is a contract assembler.
**Socratic Checkpoint:** 1. Which hook owns transport? 2. Which hook owns route/query parsing? 3. Which hook owns date params? 4. Why spread filters into the query key? 5. What hook would you copy for a new website report?

How to self-grade: Strong answers mention `useApi`, `useNavigation`, date/filter hooks, and complete query keys.
**Connects To:** Mission 12 and Mission 17, because hooks encode API contracts and types.

### Mission 11: Side Effects Are Promises to the System
**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Read side effects and predict their consequences.
**The Concept:** A side effect is a promise: when team context changes, storage updates; when auth fails, redirect happens.
**Design Intent Before You Read the Code:** Open `src/app/(main)/App.tsx:19-43` and `src/app/Providers.tsx:21-33`. Effects should synchronize with outside systems, not compute render-only values.
**Find It In The Code:** Exact files and ranges: `src/app/(main)/App.tsx:19-43`, `src/app/Providers.tsx:21-33`.

```tsx
// src/app/(main)/App.tsx:19-32
useEffect(() => {
  if (teamId) setItem(LAST_TEAM_CONFIG, teamId); // Persist last team context.
  else removeItem(LAST_TEAM_CONFIG); // Clear it when leaving team context.
}, [teamId]);

useEffect(() => {
  if (teamId && teamError) {
    removeItem(LAST_TEAM_CONFIG); // Invalid team should not stay sticky.
    router.replace('/');
  }
}, [teamId, teamError, router]);
```

```tsx
// src/app/Providers.tsx:21-33
useEffect(() => {
  document.documentElement.setAttribute('dir', dir); // External DOM side effect.
  document.documentElement.setAttribute('lang', locale); // External DOM side effect.
}, [locale, dir]);
```
**The Aha Moment:** Effects are for syncing React with things outside React render.
**Socratic Checkpoint:** 1. What side effect stores team id? 2. What side effect redirects invalid teams? 3. What side effect changes document language? 4. Why do dependencies matter? 5. What would be wrong with storing chart data in an effect?

How to self-grade: Strong answers cite effect ranges and separate sync effects from render calculations.
**Connects To:** Mission 9 and Mission 19, because side effects can create subtle bugs.

### Mission 12: The Full API Contract
**Tier:** Mid-Level
**Time Estimate:** 60 minutes
**Goal:** Map a route contract from client request to response.
**The Concept:** The API contract is the menu between dashboard and kitchen: the client orders pageviews for a date/filter range, the route promises a specific JSON shape.
**Design Intent Before You Read the Code:** Open the pageviews hook and route. The hook's request params must match the route schema and response interface.
**Find It In The Code:** Exact files and ranges: `src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-35`, `src/app/api/websites/[websiteId]/pageviews/route.ts:12-70`.

```ts
// src/components/hooks/queries/useWebsitePageviewsQuery.ts:24-32
queryFn: () =>
  get(`/websites/${websiteId}/pageviews`, {
    compare,
    startAt,
    endAt,
    unit,
    timezone,
    ...queryParams,
  }),
```

```ts
// src/app/api/websites/[websiteId]/pageviews/route.ts:55-69
return json({
  pageviews,
  sessions,
  startDate: filters.startDate,
  endDate: filters.endDate,
  compare: { pageviews: comparePageviews, sessions: compareSessions, startDate: compareStartDate, endDate: compareEndDate },
});
// Without compare, route returns { pageviews, sessions }.
```
**The Aha Moment:** Client and route are one contract split across two files.
**Socratic Checkpoint:** 1. What params does the client send? 2. What schema accepts them? 3. When is compare returned? 4. What response fields are always present? 5. Where would a type mismatch appear first?

How to self-grade: Strong answers cite `src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-35` and `src/app/api/websites/[websiteId]/pageviews/route.ts:12-70`.
**Connects To:** Mission 14 and Mission 23, because contracts need feature traces and tests.

### Mission 13: The Middleware Chain
**Tier:** Mid-Level
**Time Estimate:** 60 minutes
**Goal:** Understand request parsing, auth, and permission checks.
**The Concept:** The middleware chain is the security turnstile before analytics data leaves the building.
**Design Intent Before You Read the Code:** Open `parseRequest`, `checkAuth`, and `canViewWebsite`. The request should be validated, authenticated, and authorized before queries run.
**Find It In The Code:** Exact files and ranges: `src/lib/request.ts:12-52`, `src/lib/auth.ts:17-60`, `src/permissions/website.ts:7-40`.

```ts
// src/lib/request.ts:23-52
if (schema) {
  const result = schema.safeParse(isGet ? query : body); // Zod validates input.
  if (!result.success) error = () => badRequest(z.treeifyError(result.error));
}
if (!options?.skipAuth && !error) {
  auth = await checkAuth(request); // Auth only runs after validation succeeds.
  if (!auth) error = () => unauthorized();
}
return { url, query, body, auth, error };
```

```ts
// src/permissions/website.ts:7-40
if (user?.isAdmin) return true; // Admin override.
if (shareToken?.websiteId === websiteId || shareToken?.websiteIds?.includes(websiteId)) return true; // Share access.
const entity = await getEntity(websiteId);
if (!entity || !user) return false;
if (entity.userId) return user.id === entity.userId; // Owner access.
if (entity.teamId) return !!(await getTeamUser(entity.teamId, user.id)); // Team access.
return false;
```
**The Aha Moment:** Authentication asks "who are you"; permission asks "may you see this website."
**Socratic Checkpoint:** 1. What does `skipAuth` do? 2. What makes a share token safe? 3. Why check admin first? 4. Why fetch entity? 5. Where should a new route put permission checks?

How to self-grade: Strong answers cite validation, auth, share context, owner/team checks, and the route pattern.
**Connects To:** Mission 22 and Mission 19, because security bugs often happen at this boundary.

### Mission 14: End-to-End Feature Trace
**Tier:** Mid-Level
**Time Estimate:** 75 minutes
**Goal:** Trace the pageviews chart across the whole stack.
**The Concept:** Follow one analytics question from screen to database and back: "How many views and visitors did this website have over time?"
**Design Intent Before You Read the Code:** Read UI, hook, API, request helper, permission, query dispatch, SQL, and chart renderer. Do not skip layers.
**Find It In The Code:** Exact files and ranges: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/components/hooks/useApi.ts:20-74`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, `src/lib/request.ts:111-178`, `src/queries/sql/pageviews/getPageviewStats.ts:9-102`, `src/components/metrics/PageviewsChart.tsx:27-97`.

```text
WebsiteChart -> useWebsitePageviewsQuery -> useApi.get
  -> GET /api/websites/:websiteId/pageviews
  -> parseRequest -> canViewWebsite -> getQueryFilters
  -> getPageviewStats/getSessionStats -> json
  -> PageviewsChart
```

```ts
// src/queries/sql/pageviews/getPageviewStats.ts:9-13
export async function getPageviewStats(...args) {
  return runQuery({
    [PRISMA]: () => relationalQuery(...args), // PostgreSQL install path.
    [CLICKHOUSE]: () => clickhouseQuery(...args), // Scale-oriented analytics path.
  });
}
```
**The Aha Moment:** A feature is not a file; it is a path through responsibilities.
**Socratic Checkpoint:** 1. Where does the trace begin? 2. Where does auth happen? 3. Where do filters expand? 4. Where is DB choice made? 5. Where is the final chart dataset created?

How to self-grade: Strong answers name each layer in order with at least five cited file ranges.
**Connects To:** Mission 15 and Mission 18, because tracing prepares you to read diffs and infer decisions.

### Mission 15: Read the Diff Like an Engineer
**Tier:** Mid-Level
**Time Estimate:** 45 minutes
**Goal:** Practice reviewing a realistic analytics change.
**The Concept:** A diff is a proposed rerouting of data. Your job is to see whether the reroute preserves contracts.
**Design Intent Before You Read the Code:** Pretend a PR adds a `tag` filter to pageviews. Verify type, query key, request parsing, SQL filter mapping, and permission.
**Find It In The Code:** Exact files and ranges: `src/lib/types.ts:83-105`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-32`, `src/lib/request.ts:79-90`, `src/lib/prisma.ts:74-150`, `src/lib/clickhouse.ts:73-140`.

```ts
// src/lib/types.ts:83-105
export interface FilterParams {
  path?: string;
  referrer?: string;
  tag?: string; // The type already knows tag can be a filter.
  eventType?: number;
  match?: 'all' | 'any';
}
```

```ts
// src/lib/request.ts:79-90
for (const key of Object.keys(query)) {
  const baseName = key.replace(/\d+$/, '');
  if (baseName in FILTER_COLUMNS) {
    result[key] = query[key]; // Only recognized filter columns survive.
  }
}
```
**The Aha Moment:** A good review follows the data contract, not just changed lines.
**Socratic Checkpoint:** 1. Does the filter affect cache key? 2. Does the route preserve it? 3. Does SQL map it safely? 4. Does ClickHouse match Prisma behavior? 5. Is authorization unchanged?

How to self-grade: Strong answers cite hook key, request filter extraction, Prisma mapper, ClickHouse mapper, and route permission.
**Connects To:** Mission 19 and Mission 22, because review finds bugs and security drift early.

### Mission 16: Composition Over Inheritance
**Tier:** Mid-Level
**Time Estimate:** 40 minutes
**Goal:** See how UI behavior is built by combining components and hooks.
**The Concept:** The dashboard is assembled from reusable panels like analytics instruments mounted on a board.
**Design Intent Before You Read the Code:** Open `WebsiteMetricsBar`, `MetricCard`, and `PageviewsChart`. The system composes small parts rather than subclassing components.
**Find It In The Code:** Exact files and ranges: `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:25-90`, `src/components/metrics/MetricCard.tsx:9-72`, `src/components/metrics/PageviewsChart.tsx:21-97`.

```tsx
// src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:25-64
const metrics = data ? [
  { value: visitors, label: t(labels.visitors), change: visitors - comparison.visitors, formatValue: formatLongNumber },
  { value: pageviews, label: t(labels.views), change: pageviews - comparison.pageviews, formatValue: formatLongNumber },
] : null;
// Data becomes a config array.
```

```tsx
// src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:74-89
<MetricsBar>
  {metrics?.map(({ label, value, change, formatValue }) => (
    <MetricCard key={label} value={value} label={label} change={change} formatValue={formatValue} showChange={!isAllTime} />
  ))}
</MetricsBar>
```
**The Aha Moment:** Composition lets one display primitive serve many metrics.
**Socratic Checkpoint:** 1. What is the config array? 2. What does `MetricCard` not know? 3. Why is `formatValue` compositional? 4. How does `LoadingPanel` compose state? 5. What would inheritance add here?

How to self-grade: Strong answers explain that data shaping stays in parent and display stays in child.
**Connects To:** Mission 4 and Mission 21, because composition affects performance and re-render boundaries.

### Mission 17: TypeScript's Hidden Work
**Tier:** Mid-Level
**Time Estimate:** 50 minutes
**Goal:** Notice the invisible constraints TypeScript gives reviewers.
**The Concept:** TypeScript is the quiet auditor checking report columns before the dashboard opens.
**Design Intent Before You Read the Code:** Open `src/lib/types.ts` and a query hook. Find where types protect shape and where `any` bypasses protection.
**Find It In The Code:** Exact files and ranges: `src/lib/types.ts:1-13`, `src/lib/types.ts:122-193`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-35`, `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:21-46`.

```ts
// src/lib/types.ts:122-130
export interface PageResult<T> {
  data: T; // Generic lets paged queries keep item type.
  count: number;
  page: number;
  pageSize: number;
}
```

```tsx
// src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:21-46
const { pageviews, sessions, compare } = (data || {}) as any;
// This bypasses the WebsitePageviewsData interface and hides compare-shape uncertainty.
```
**The Aha Moment:** `any` is a place where the compiler stops helping.
**Socratic Checkpoint:** 1. What does `ReactQueryOptions<T>` protect? 2. Where is `Auth` typed? 3. Where does `any` weaken chart safety? 4. What type would you add for compare data? 5. Why do generic page results matter?

How to self-grade: Strong answers cite `src/lib/types.ts:1-31`, `src/lib/types.ts:122-193`, and the `any` cast in `WebsiteChart`.
**Connects To:** Mission 23 and Mission 18, because tests and architecture decisions often compensate for weak types.

