# Journal

## Mission Design Rationale

The missions are ordered from heartbeat to ownership. Early missions teach where the app starts and what folders mean. Middle missions teach state, hooks, effects, API contracts, middleware-like parsing, and end-to-end tracing. Senior missions train judgment: bug prediction, performance, security, missing tests, and history reading.

## Why This Order

The app shell appears before individual features because `src/app/layout.tsx:1-40`, `src/app/Providers.tsx:1-62`, and `src/app/(main)/App.tsx:13-92` explain how every page runs. Hooks appear before API contracts because UI data flow begins in hooks such as `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`. Persistence appears after route tracing because the route decides whether a request is allowed before query modules run (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-33`).

## Code Paths Chosen

- Website pageviews chart: compact end-to-end analytics flow (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`).
- Website creation: real CRUD-style mutation flow (`src/app/(main)/websites/WebsiteAddForm.tsx:14-59`, `src/app/api/websites/route.ts:38-99`, `src/queries/prisma/website.ts:113-119`).
- Auth and permissions: critical safety boundary (`src/lib/auth.ts:17-60`, `src/permissions/website.ts:7-40`).
- DB dispatch: core architecture choice (`src/lib/db.ts:22-36`, `src/queries/sql/pageviews/getPageviewStats.ts:9-102`).

## Skills By Tier

- Junior: locate entry points, read props, follow one component, identify routes, understand basic TypeScript contracts.
- Mid-Level: trace state ownership, custom hooks, side effects, route contracts, filter expansion, and feature flows.
- Senior: critique architecture, predict bugs, audit security/performance, write tests for risk, and read evolution from commits.

## What A Senior Would Do Differently

A senior reads for invariants. They ask whether every route follows `parseRequest -> permission -> query -> response` (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-70`), whether every hook includes complete cache keys (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`), whether every analytics query maintains Prisma/ClickHouse parity (`src/queries/sql/pageviews/getPageviewStats.ts:9-102`), and whether user-visible math handles empty data (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`).

