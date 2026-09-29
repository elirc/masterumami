# Tier 1: Junior Missions

### Mission 1: The App's Heartbeat
**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** Find how the app starts and what wraps every page.
**The Concept:** Every analytics dashboard needs a control room power switch. In Umami, root layout and providers are that switch: fonts, metadata, i18n, routing, query cache, and error boundaries start there.
**Design Intent Before You Read the Code:** Open `src/app/layout.tsx:1-40` and `src/app/Providers.tsx:1-62`. The root layout owns server-level document structure. `Providers` owns client-level runtime services. Poor implementation here would make every page fail at once.
**Find It In The Code:** Exact files and ranges: `src/app/layout.tsx:1-40`, `src/app/Providers.tsx:11-62`.

```tsx
// src/app/layout.tsx:14-37
export default function ({ children }) {
  if (process.env.DISABLE_UI) {
    // Environment-level shutoff: no routes render visible UI.
    return <html><body></body></html>;
  }

  return (
    <html lang="en" className={`${inter.className} ${inter.variable}`}>
      <body>
        <Suspense>
          <Providers>{children}</Providers>
          {/* Every page gets the client runtime below Providers. */}
        </Suspense>
      </body>
    </html>
  );
}
```

```tsx
// src/app/Providers.tsx:11-19 and 51-61
const client = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false, // Failed queries surface quickly instead of silently retrying.
      refetchOnWindowFocus: false, // Dashboards do not refresh just because a tab regains focus.
      staleTime: 1000 * 60, // Server data is fresh for one minute.
    },
  },
});

return (
  <ZenProvider>
    <RouterProvider navigate={navigate}>
      <MessagesProvider>
        <QueryClientProvider client={client}>
          <ErrorBoundary>{children}</ErrorBoundary>
        </QueryClientProvider>
      </MessagesProvider>
    </RouterProvider>
  </ZenProvider>
);
```
**The Aha Moment:** The app is not "just pages"; every page is inside a runtime stack.
**Socratic Checkpoint:**
1. Which file renders `<html>` and `<body>`?
2. Which file creates the React Query client?
3. Why is `Providers` a client component?
4. What happens when `DISABLE_UI` is set?
5. Where would a global error boundary catch page errors?

How to self-grade: Strong answers cite `src/app/layout.tsx:14-37` and `src/app/Providers.tsx:11-62`, and explain that server document setup and client provider setup are separate responsibilities.
**Connects To:** Mission 2 and Mission 8, because folder layout and navigation only make sense after you know the wrapper.

### Mission 2: The Folder Mental Map
**Tier:** Junior
**Time Estimate:** 40 minutes
**Goal:** Learn what the major folders own.
**The Concept:** Think of Umami like an analytics office: `app` is the front desk and API counter, `components` are reusable tools, `lib` is shared policy, `queries` is the records room, and `prisma` is the filing cabinet blueprint.
**Design Intent Before You Read the Code:** Use `rg --files` and then open representative files. `src/app` should contain pages/routes, `src/components` should contain UI/hooks, `src/lib` should contain shared helpers, and `src/queries` should touch data. If these responsibilities blur, changes become hard to review.
**Find It In The Code:** Exact files and ranges: `src/app/(main)/layout.tsx:5-10`, `src/components/hooks/useApi.ts:20-74`, `src/lib/request.ts:12-52`, `src/queries/prisma/website.ts:7-119`, `prisma/schema.prisma:66-97`.

```tsx
// src/app/(main)/layout.tsx:5-10
export default function ({ children }) {
  return (
    <Suspense>
      <App>{children}</App>
      {/* The route group delegates shell behavior to App. */}
    </Suspense>
  );
}
```

```ts
// src/queries/prisma/website.ts:113-119
export async function createWebsite(data) {
  return prisma.client.website.create({
    data, // Persistence layer receives already-authorized, already-shaped data.
  });
}
```
**The Aha Moment:** Folders are ownership boundaries, not storage bins.
**Socratic Checkpoint:**
1. Which folder owns API handlers?
2. Which folder owns Prisma entity functions?
3. Which folder owns reusable hooks?
4. Which folder defines database tables?
5. What folder would you inspect for auth helpers?

How to self-grade: Strong answers name `src/app/api`, `src/queries/prisma`, `src/components/hooks`, `prisma/schema.prisma`, and `src/lib/auth.ts:17-60`.
**Connects To:** Mission 3 and Mission 5, because contracts and routes rely on folder ownership.

### Mission 3: TypeScript Is a Contract
**Tier:** Junior
**Time Estimate:** 45 minutes
**Goal:** Understand how TypeScript describes app contracts.
**The Concept:** Analytics data is like a report template: the chart can only render correctly if the data contract says what columns exist.
**Design Intent Before You Read the Code:** Open `src/lib/types.ts:1-120` and `src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-35`. Types should describe auth, filters, dates, and API response shapes. Weak types let bad data travel too far.
**Find It In The Code:** Exact files and ranges: `src/lib/types.ts:6-31`, `src/lib/types.ts:66-120`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-13`.

```ts
// src/lib/types.ts:6-13
export type ObjectValues<T> = T[keyof T]; // Turns a constants object into a union of its values.
export type Role = ObjectValues<typeof ROLES>; // Role stays tied to the constants source of truth.
export type Operator = (typeof OPERATORS)[keyof typeof OPERATORS]; // Filters accept known operators only.
```

```ts
// src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-13
export interface WebsitePageviewsData {
  pageviews: { x: string; y: number }[]; // Time bucket plus count for pageviews.
  sessions: { x: string; y: number }[]; // Time bucket plus count for sessions.
}

export function useWebsitePageviewsQuery(
  { websiteId, compare }: { websiteId: string; compare?: string },
  options?: ReactQueryOptions<WebsitePageviewsData>,
) { /* hook body owns the fetch contract */ }
```
**The Aha Moment:** Types are how distant files agree without talking at runtime.
**Socratic Checkpoint:**
1. What does `ObjectValues<T>` do?
2. Why does `WebsitePageviewsData` include arrays?
3. Where is `Auth` defined?
4. Why are filter params composed from several interfaces?
5. What type prevents callers from replacing a hook's query key?

How to self-grade: Strong answers cite `src/lib/types.ts:6-31`, `src/lib/types.ts:66-120`, and `src/lib/types.ts:1-9`.
**Connects To:** Mission 6 and Mission 17, because props and hidden type work are contracts too.

### Mission 4: Your First React Component
**Tier:** Junior
**Time Estimate:** 35 minutes
**Goal:** Read a small component end to end.
**The Concept:** A metric card is like one tile on the analytics wall: label, number, optional change, and formatting.
**Design Intent Before You Read the Code:** Open `src/components/metrics/MetricCard.tsx:9-72`. Props define input. Derived values compute display state. JSX renders the visual structure. A poor component would mix fetching with display.
**Find It In The Code:** Exact file and range: `src/components/metrics/MetricCard.tsx:9-72`.

```tsx
// src/components/metrics/MetricCard.tsx:21-34
export const MetricCard = ({ value = 0, change = 0, formatValue = formatNumber, showChange = false }) => {
  const diff = value - change; // Reconstructs previous value from current value and delta.
  const pct = diff !== 0 ? ((value - diff) / diff) * 100 : value !== 0 ? 100 : 0;
  const props = useSpring({ x: Number(value) || 0, from: { x: 0 } }); // Animates current value.
  const changeProps = useSpring({ x: Number(pct) || 0, from: { x: 0 } }); // Animates percent change.
```

```tsx
// src/components/metrics/MetricCard.tsx:36-72
return (
  <Column>
    {showLabel && <Text weight="bold">{label}</Text>}
    <Text size="4xl" weight="bold">
      <AnimatedDiv>{props?.x?.to(x => formatValue(x))}</AnimatedDiv>
    </Text>
    {showChange && (
      <ChangeLabel value={change} reverseColors={reverseColors}>
        <AnimatedDiv>{changeProps?.x?.to(x => `${Math.abs(~~x)}%`)}</AnimatedDiv>
      </ChangeLabel>
    )}
  </Column>
);
```
**The Aha Moment:** A simple component should transform props into UI, not know where data came from.
**Socratic Checkpoint:**
1. Which props have defaults?
2. What does `useSpring` animate?
3. Why is `formatValue` passed as a prop?
4. When does `ChangeLabel` render?
5. What would break if `value` were a string?

How to self-grade: Strong answers cite `src/components/metrics/MetricCard.tsx:9-34` and `src/components/metrics/MetricCard.tsx:63-70`.
**Connects To:** Mission 6 and Mission 16, because component props enable composition.

### Mission 5: Your First Node.js Route
**Tier:** Junior
**Time Estimate:** 45 minutes
**Goal:** Read one API route from request to response.
**The Concept:** An API route is the guarded desk where the dashboard asks for data. It checks ID, credentials, and request shape before touching analytics records.
**Design Intent Before You Read the Code:** Open `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`. The route should validate, authenticate, authorize, query, and respond. If it queries before authorization, private analytics can leak.
**Find It In The Code:** Exact file and range: `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`.

```ts
// src/app/api/websites/[websiteId]/pageviews/route.ts:12-28
const schema = withDateRange({ ...filterParams }); // Defines accepted query params.
const { auth, query, error } = await parseRequest(request, schema); // Validates and authenticates.
if (error) return error(); // Stops invalid or unauthenticated requests.
const { websiteId } = await params; // Reads the dynamic URL segment.
if (!(await canViewWebsite(auth, websiteId))) return unauthorized(); // Enforces domain access.
```

```ts
// src/app/api/websites/[websiteId]/pageviews/route.ts:28-70
const filters = await getQueryFilters(query, websiteId); // Converts URL params to query filters.
const [pageviews, sessions] = await Promise.all([
  getPageviewStats(websiteId, filters), // Pageview time series.
  getSessionStats(websiteId, filters), // Session time series.
]);
return json({ pageviews, sessions }); // Client hook expects this shape.
```
**The Aha Moment:** A route is a sequence of gates before a database call.
**Socratic Checkpoint:**
1. What validates query params?
2. What checks authentication?
3. What checks authorization?
4. Why use `Promise.all`?
5. What response shape reaches the chart?

How to self-grade: Strong answers cite `src/lib/request.ts:12-52`, `src/permissions/website.ts:7-40`, and `src/app/api/websites/[websiteId]/pageviews/route.ts:30-70`.
**Connects To:** Mission 12 and Mission 13, because routes are API contracts plus middleware chain.

### Mission 6: Props Are a Typed Contract
**Tier:** Junior
**Time Estimate:** 30 minutes
**Goal:** See how parent components configure child components.
**The Concept:** Props are like the dashboard work order: the parent says which website and mode, the child decides how to render.
**Design Intent Before You Read the Code:** Open `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:7-20` and `src/components/metrics/PageviewsChart.tsx:9-21`. Props should identify required inputs and optional behavior.
**Find It In The Code:** Exact files and ranges: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:7-20`, `src/components/metrics/PageviewsChart.tsx:9-21`, `src/components/metrics/MetricCard.tsx:9-19`.

```tsx
// src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:7-13
export function WebsiteChart({
  websiteId, // Required: tells the hook which tracked property to fetch.
  compareMode, // Optional: decides whether to request comparison data.
}: {
  websiteId: string;
  compareMode?: boolean;
}) { /* component body */ }
```

```ts
// src/components/metrics/PageviewsChart.tsx:9-19
export interface PageviewsChartProps extends BarChartProps {
  data: {
    pageviews: any[]; // Series from API route.
    sessions: any[]; // Series from API route.
    compare?: { pageviews: any[]; sessions: any[] }; // Optional previous period.
  };
  unit: string; // Date grouping like hour/day/month.
}
```
**The Aha Moment:** Props describe what a component needs, not how the parent found it.
**Socratic Checkpoint:**
1. Which prop is required for `WebsiteChart`?
2. Which prop is optional?
3. What does `PageviewsChart` require in `data`?
4. Why does `MetricCard` accept `formatValue`?
5. Where would you add a new display-only prop?

How to self-grade: Strong answers cite the exact prop definitions and avoid describing unrelated hooks.
**Connects To:** Mission 4 and Mission 14, because full-stack data eventually becomes component props.

### Mission 7: Following Data Into the App
**Tier:** Junior
**Time Estimate:** 50 minutes
**Goal:** Trace data from hook to component rendering.
**The Concept:** Analytics data enters like a report packet: fetched once, reshaped, then handed to charts.
**Design Intent Before You Read the Code:** Open `WebsiteChart`, `useWebsitePageviewsQuery`, and `PageviewsChart`. Data fetching should live in the hook, not the chart renderer.
**Find It In The Code:** Exact files and ranges: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/components/metrics/PageviewsChart.tsx:27-97`.

```tsx
// src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:17-21
const { data, isLoading, isFetching, error } = useWebsitePageviewsQuery({ websiteId, compare });
const { pageviews, sessions, compare } = (data || {}) as any;
// Data enters the component through one hook call.
```

```tsx
// src/components/metrics/PageviewsChart.tsx:27-52
const chartData: any = useMemo(() => {
  if (!data) return; // No chart config until data exists.
  return {
    datasets: [
      { type: 'bar', label: t(labels.visitors), data: generateTimeSeries(data.sessions, minDate, maxDate, unit, dateLocale) },
      { type: 'bar', label: t(labels.views), data: generateTimeSeries(data.pageviews, minDate, maxDate, unit, dateLocale) },
    ],
  };
}, [data, locale]);
```
**The Aha Moment:** Fetching and chart rendering are two different jobs.
**Socratic Checkpoint:**
1. Which file fetches data?
2. Which file turns data into chart datasets?
3. Why does `LoadingPanel` wrap the chart?
4. What happens when `data` is undefined?
5. Where is comparison data aligned?

How to self-grade: Strong answers cite `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:23-57` and `src/components/metrics/PageviewsChart.tsx:27-83`.
**Connects To:** Mission 9 and Mission 14, because data flow becomes state and full-stack tracing.

### Mission 8: Navigation Is the App's Skeleton
**Tier:** Junior
**Time Estimate:** 40 minutes
**Goal:** Understand how IDs and query params are read from the URL.
**The Concept:** Navigation is the dashboard's skeleton: website pages, team pages, and filters hang off URL bones.
**Design Intent Before You Read the Code:** Open `src/components/hooks/useNavigation.ts:5-58` and `src/app/(main)/App.tsx:13-32`. The hook should derive ids and query params from Next navigation. Poor URL parsing makes pages show the wrong account or website.
**Find It In The Code:** Exact files and ranges: `src/components/hooks/useNavigation.ts:5-58`, `src/app/(main)/App.tsx:13-32`.

```ts
// src/components/hooks/useNavigation.ts:5-14
const router = useRouter();
const pathname = usePathname();
const searchParams = useSearchParams();
const [, teamId] = pathname.match(/\/teams\/([a-f0-9-]+)/) || [];
const [, websiteId] = pathname.match(/\/websites\/([a-f0-9-]+)/) || [];
// IDs are derived from URL patterns.
```

```ts
// src/components/hooks/useNavigation.ts:44-57
return {
  router,
  pathname,
  query: queryParams, // Plain object version of search params.
  teamId,
  websiteId,
  updateParams, // Keep existing query params and merge changes.
  replaceParams, // Replace query params entirely.
  renderUrl, // Prefix team context when needed.
};
```
**The Aha Moment:** URL state is application state.
**Socratic Checkpoint:**
1. How is `websiteId` extracted?
2. What is `queryParams`?
3. Why does `renderUrl` know about `teamId`?
4. Where does `App` store last team id?
5. What could break if URL regexes change?

How to self-grade: Strong answers cite `src/components/hooks/useNavigation.ts:9-18`, `src/components/hooks/useNavigation.ts:30-57`, and `src/app/(main)/App.tsx:19-32`.
**Connects To:** Mission 9 and Mission 11, because URL state drives side effects and server data.

