# Mid-Level Engineer Guide

## Full-Stack Architecture Diagram

```text
Browser
  |
  | React components under src/app/(main) and src/components
  v
Client providers: Zen UI, Router adapter, i18n, React Query, ErrorBoundary
  src/app/Providers.tsx:51-61
  |
  | useApi adds auth/share headers and /api base path
  v
Next App Router API handlers under src/app/api
  src/app/api/websites/[websiteId]/pageviews/route.ts:8-70
  |
  | parseRequest + permissions + filters
  v
Query modules
  src/queries/sql/pageviews/getPageviewStats.ts:9-102
  src/queries/prisma/website.ts:7-119
  |
  | runQuery selects ClickHouse when CLICKHOUSE_URL exists, else Prisma/PostgreSQL
  v
PostgreSQL / ClickHouse
  prisma/schema.prisma:66-150
```

## TypeScript Type System Deep Dive

The codebase uses small generic utilities to turn constants into domain types. `ObjectValues<T>` becomes the foundation for `TimeUnit`, `Role`, `DynamicDataType`, and `Operator` (`src/lib/types.ts:6-13`). This pattern keeps constants in `src/lib/constants.ts` authoritative while allowing TypeScript to reason about values.

Query filters are compositional: `QueryFilters` extends date, filter, sort, paging, and segment interfaces (`src/lib/types.ts:66-120`). That shape matches how `getQueryFilters` merges parsed date range, URL filters, segment filters, cohort filters, paging, sort, search, and comparison into one object (`src/lib/request.ts:111-178`).

React Query hooks accept `ReactQueryOptions<T>` but exclude `queryKey` and `queryFn` (`src/lib/types.ts:1-9`). That is intentional: a feature hook like `useWebsitePageviewsQuery` owns its cache key and fetch URL (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`), while callers can still pass options such as `enabled`, `staleTime`, or selection behavior.

## State Management Deep Dive

Server data lives in TanStack Query. `Providers` creates a `QueryClient` with no retries, no refetch on focus, and one-minute stale time (`src/app/Providers.tsx:11-19`). Feature hooks call `useApi().useQuery`, as `useWebsitePageviewsQuery` does (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:15-35`).

App preferences and identity-like local state live in Zustand. `src/store/app.ts` initializes locale, theme, timezone, date range, share data, user, and config (`src/store/app.ts:14-23`). It exposes setter functions that call `store.setState` (`src/store/app.ts:27-54`).

The date range flow is mixed URL plus local storage. `useDateRange` reads `date`, `unit`, `offset`, and `compare` from navigation query params (`src/components/hooks/useDateRange.ts:8-12`), falls back to stored/default date range (`src/components/hooks/useDateRange.ts:13-24`), and computes comparison dates (`src/components/hooks/useDateRange.ts:26-37`).

## API Contract Map

- `GET /api/websites/:websiteId/pageviews`: query params are date range plus filters (`src/app/api/websites/[websiteId]/pageviews/route.ts:12-16`); response is `{ pageviews, sessions }` or includes `compare` when requested (`src/app/api/websites/[websiteId]/pageviews/route.ts:55-69`).
- `GET /api/websites`: paging/search/includeTeams params (`src/app/api/websites/route.ts:14-19`); response is user websites or user plus team-owned websites (`src/app/api/websites/route.ts:27-35`).
- `POST /api/websites`: body contains `name`, `domain`, optional `shareId`, `teamId`, `id` (`src/app/api/websites/route.ts:38-47`); route creates website and optional share (`src/app/api/websites/route.ts:71-99`).
- `POST /api/auth/login`: body contains `username` and `password` (`src/app/api/auth/login/route.ts:12-18`); response contains token and user object (`src/app/api/auth/login/route.ts:42-47`).

## Component Interaction Map

`WebsiteChart` depends on timezone/date hooks, a pageviews query hook, and `PageviewsChart` (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:1-6`). `PageviewsChart` depends on theme, locale, labels, colors, date-series generation, and `BarChart` (`src/components/metrics/PageviewsChart.tsx:1-8`). `MetricCard` is a lower-level display primitive used by `WebsiteMetricsBar` to render visitors, visits, views, bounce rate, and duration (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:25-90`).

## Full-Stack Feature Trace: Website Pageviews Chart

1. UI reads timezone and date range. `WebsiteChart` calls `useTimezone` and `useDateRange` (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-16`).
2. UI starts a typed query. `WebsiteChart` calls `useWebsitePageviewsQuery` with `websiteId` and optional compare string (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:17-20`).
3. Hook builds cache key and URL. Query key includes website, compare, date, timezone, and filters (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-23`), and query function calls `/websites/${websiteId}/pageviews` with the same params (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:24-32`).
4. API client adds auth and `/api`. `useApi` creates authorization and share headers (`src/components/hooks/useApi.ts:20-33`) and prefixes relative URLs with `/api` plus base path (`src/components/hooks/useApi.ts:34-38`).
5. Route validates and authenticates. `parseRequest` validates schema and checks auth unless skipped (`src/lib/request.ts:12-52`).
6. Route authorizes website access. `canViewWebsite` accepts admins, valid share tokens, direct owner, or team member (`src/permissions/website.ts:7-40`).
7. Route expands filters. `getQueryFilters` builds date range, base filters, segment/cohort filters, and compare/search/page fields (`src/lib/request.ts:111-178`).
8. Route runs analytics queries. Pageviews and sessions are fetched in parallel (`src/app/api/websites/[websiteId]/pageviews/route.ts:30-33`), with optional comparison queries (`src/app/api/websites/[websiteId]/pageviews/route.ts:35-66`).
9. Query dispatches to storage. `getPageviewStats` chooses Prisma or ClickHouse implementation through `runQuery` (`src/queries/sql/pageviews/getPageviewStats.ts:9-13`, `src/lib/db.ts:22-36`).
10. UI renders. `WebsiteChart` aligns comparison points (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:23-46`) and passes data to `PageviewsChart` (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:48-57`).

## Diff Reading Exercise

Hypothetical change: add a `host` filter to the pageviews chart.

Read these files in order:

1. `src/lib/types.ts:83-105` to confirm `host` is already a filter field.
2. `src/lib/request.ts:79-90` to confirm query params with names in `FILTER_COLUMNS` are preserved.
3. `src/lib/prisma.ts:74-150` and `src/lib/clickhouse.ts:73-140` to verify filter clauses map safely.
4. `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-32` to verify all filter params are included in key and request.

Review questions:

- Does the diff add the filter to both UI state and query string?
- Does the query key change when the filter changes?
- Does Prisma and ClickHouse both know how to map the filter?
- Does the route still call `canViewWebsite` before querying?

## Non-Obvious Architectural Patterns

- Dual query implementations: analytics queries expose one function and hide storage choice behind `runQuery` (`src/queries/sql/pageviews/getPageviewStats.ts:9-13`, `src/lib/db.ts:22-36`). This lets production ClickHouse scale while PostgreSQL remains the base install path.
- URL-driven analytics state: `useNavigation` exposes query params and URL builders (`src/components/hooks/useNavigation.ts:14-58`), and date/filter hooks consume those params (`src/components/hooks/useDateRange.ts:8-37`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:15-32`). This makes dashboard views shareable.
- Permission functions are domain-specific. Website permissions live in `src/permissions/website.ts:7-152`, which keeps route handlers short and reviewable.
- Raw SQL is centralized. Prisma raw queries use placeholder replacement in one helper (`src/lib/prisma.ts:255-282`), and ClickHouse queries use official query params (`src/lib/clickhouse.ts:265-287`).

## Mid-Level Socratic Checkpoint

1. Why does `useWebsitePageviewsQuery` include date/filter params in both query key and request params?
2. What breaks if `canViewWebsite` is removed from the route?
3. Why does `getPageviewStats` have two SQL implementations?
4. How does a segment change SQL without the chart knowing about segments?
5. Which state belongs in Zustand and which belongs in React Query?
6. What is the difference between `pageviews` and `sessions` in the chart response?
7. How would you review a new analytics filter?
8. Why is ClickHouse allowed to use hourly rollups in some cases but raw events in others?

## How To Self-Grade

A strong answer references `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`, `src/permissions/website.ts:7-40`, `src/queries/sql/pageviews/getPageviewStats.ts:59-99`, `src/lib/request.ts:123-160`, `src/store/app.ts:14-54`, `src/app/Providers.tsx:11-19`, and `src/app/api/websites/[websiteId]/pageviews/route.ts:30-69`. It should explain tradeoffs, not just name files.

