# Reading Map

## Mental Model

Umami is an analytics control room: the browser asks for a website dashboard, client hooks translate the selected date range and filters into API calls, Next route handlers authenticate and authorize the request, SQL query modules summarize event/session data from PostgreSQL or ClickHouse, and React chart/table components render the response back into the dashboard (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, `src/queries/sql/pageviews/getPageviewStats.ts:9-102`).

## Top 10 Files To Read In Order

1. `package.json:13-49`
   Why now: scripts reveal the project lifecycle. Understand: dev, build, db, test, lint, seed. Explain after reading: why build is more than `next build`.

2. `README.md:28-69`
   Why now: it states source install requirements and base setup. Understand: Node, PostgreSQL, `.env`, build, start. Explain after reading: what is documented and what local dev still requires you to infer.

3. `src/app/layout.tsx:1-40`
   Why now: root App Router layout starts all pages. Understand: font, metadata, UI disable flag, providers. Explain after reading: why `Providers` is inside `Suspense`.

4. `src/app/Providers.tsx:1-62`
   Why now: client runtime dependencies live here. Understand: QueryClient defaults, Zen UI, router adapter, i18n provider, error boundary. Explain after reading: where React Query enters the app.

5. `src/app/(main)/App.tsx:13-92`
   Why now: authenticated product shell. Understand: login query, config, team loading, redirects, navigation, self-tracking scripts. Explain after reading: how unauthenticated users leave the main app.

6. `src/components/hooks/useApi.ts:20-74`
   Why now: most client-server calls pass through this hook. Understand: auth headers, share headers, base path, response handling, query/mutation export. Explain after reading: how `/websites/x/pageviews` becomes `/api/websites/x/pageviews`.

7. `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`
   Why now: a small typed data-fetching hook shows the app pattern. Understand: query key, date/filter params, enabled guard. Explain after reading: why query keys include filters.

8. `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`
   Why now: it is a complete backend route anatomy. Understand: Zod schema, `parseRequest`, permission, query filters, parallel queries, comparison response. Explain after reading: where authorization happens.

9. `src/queries/sql/pageviews/getPageviewStats.ts:9-102`
   Why now: it reveals the analytics query architecture. Understand: PostgreSQL path, ClickHouse path, raw rows vs hourly rollups. Explain after reading: when ClickHouse avoids scanning raw events.

10. `prisma/schema.prisma:66-150`
    Why now: product data vocabulary becomes concrete. Understand: `Website`, `Session`, `WebsiteEvent`, indexes. Explain after reading: why analytics tables are indexed by `websiteId` and time.

## Three Most Important Data Flows

1. Dashboard chart data: `WebsiteChart` reads date range and calls `useWebsitePageviewsQuery` (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-20`), the hook calls `GET /api/websites/:websiteId/pageviews` (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:24-32`), the route authorizes and runs pageview/session queries (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-33`), and the chart renders time-series data (`src/components/metrics/PageviewsChart.tsx:27-97`).

2. Authenticated shell flow: main app calls `useLoginQuery`, loads config and optional team state (`src/app/(main)/App.tsx:13-18`), redirects on auth error (`src/app/(main)/App.tsx:38-43`), then renders nav plus children (`src/app/(main)/App.tsx:53-90`).

3. Website creation flow: form validates name/domain and posts through `useUpdateQuery` (`src/app/(main)/websites/WebsiteAddForm.tsx:14-24`), route validates body and permission (`src/app/api/websites/route.ts:38-69`), creates the website and optional share (`src/app/api/websites/route.ts:71-99`), and persistence uses Prisma `website.create` (`src/queries/prisma/website.ts:113-119`).

## Pre-Reading Checklist

1. Can you explain why `src/app/(main)` does not appear in URLs?
2. Can you identify whether a file is a client component by looking for `'use client'`?
3. Can you distinguish React Query state from Zustand state?
4. Can you find where bearer tokens are attached to API requests?
5. Can you find where API handlers reject invalid query/body input?
6. Can you explain why analytics filters must affect both query keys and SQL?
7. Can you map `websiteId` from URL to route param to SQL parameter?
8. Can you tell whether a query is PostgreSQL/Prisma or ClickHouse?
9. Can you explain the difference between `Session` and `WebsiteEvent`?
10. Can you find one test runner config and one e2e test folder?

## Red Flags Checklist

- A new query hook that omits a filter from its query key, unlike `useWebsitePageviewsQuery` (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`).
- An API route that skips `parseRequest` or authorization checks, unlike the pageviews route (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-28`).
- Raw SQL dynamic column names built from user input instead of constants or parsed filters (`src/lib/prisma.ts:74-150`, `src/lib/clickhouse.ts:73-140`).
- New analytics code that only implements Prisma or only ClickHouse when it should support both through `runQuery` (`src/lib/db.ts:22-36`).
- Client code assuming every response succeeds instead of handling `handleResponse` rejection (`src/components/hooks/useApi.ts:11-18`).
- State that belongs in URL/search params stored only in component local state, bypassing `useNavigation` and date/filter hooks (`src/components/hooks/useNavigation.ts:5-58`).
- Metrics that divide by zero, as `WebsiteMetricsBar` currently risks when `visits` is zero (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`).
- New environment requirements that are not checked by `scripts/check-env.js` (`scripts/check-env.js:21-27`).
- New tables without indexes shaped around website/time access, unlike `WebsiteEvent` indexes (`prisma/schema.prisma:135-149`).

