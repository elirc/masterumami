# Reference Suite

## Doc 1: Junior Onboarding Guide

Start with setup: Node.js 18.18+ and PostgreSQL v12.14+ are required (`README.md:28-31`), install with `pnpm install` (`README.md:33-39`), create `.env` with `DATABASE_URL` (`README.md:41-53`), then use scripts from `package.json:13-49`. For local learning, run database generation/migration, seed demo analytics data, and start dev mode (`package.json:29-35`, `package.json:46-46`, `package.json:14-14`).

First files to open:

- `src/app/layout.tsx:1-40`: root page shell.
- `src/app/Providers.tsx:1-62`: client providers.
- `src/app/(main)/App.tsx:13-92`: authenticated app shell.
- `src/components/hooks/useApi.ts:20-74`: API wrapper.
- `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`: route pattern.
- `prisma/schema.prisma:66-150`: analytics domain tables.

## Doc 2: Mid-Level Architecture Guide

The system separates UI, client data fetching, API route handling, authorization, query orchestration, and persistence. React Query is configured globally in `src/app/Providers.tsx:11-19`. Feature hooks use `useApi` to call `/api` endpoints with bearer/share headers (`src/components/hooks/useApi.ts:20-74`). API routes call `parseRequest` for validation/auth (`src/lib/request.ts:12-52`) and permission helpers for domain access (`src/permissions/website.ts:7-40`). Query modules call `runQuery`, which chooses ClickHouse when `CLICKHOUSE_URL` is set and Prisma/PostgreSQL otherwise (`src/lib/db.ts:22-36`).

Key contract: if you add a new analytics capability, update the UI hook, API route schema, permission check, query filter handling, Prisma query path, and ClickHouse query path when needed (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, `src/queries/sql/pageviews/getPageviewStats.ts:9-102`).

## Doc 3: Senior Ownership Guide

Own these risks:

- Dual-storage parity: Prisma and ClickHouse query paths must agree (`src/queries/sql/pageviews/getPageviewStats.ts:16-102`).
- Raw SQL safety: Prisma placeholders are custom-replaced (`src/lib/prisma.ts:255-282`) and ClickHouse uses query params (`src/lib/clickhouse.ts:265-287`).
- Auth boundaries: normal bearer auth and share-token auth differ (`src/lib/auth.ts:17-60`).
- Metrics correctness: zero-division and compare alignment need guardrails (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`, `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:31-43`).
- Setup drift: scripts and README should stay aligned (`README.md:28-69`, `package.json:13-49`).

## Doc 4: Code Review Guide

Review in this order:

1. Entry point: Is the route/component in the expected App Router location? Compare `src/app/(main)/layout.tsx:5-10` and `src/app/api/websites/[websiteId]/pageviews/route.ts:8-11`.
2. Validation: Does the route define a schema and call `parseRequest`? See `src/app/api/websites/route.ts:38-50`.
3. Authorization: Does the route call a permission helper before data access? See `src/app/api/websites/[websiteId]/pageviews/route.ts:22-28`.
4. Query keys: Does the hook include all params that affect data? See `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-23`.
5. Storage path: Does the query support the relevant DB mode? See `src/lib/db.ts:22-36`.
6. UI edge cases: Are loading/error/empty states handled through `LoadingPanel` or equivalent? See `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:48-57`.
7. Types: Does the diff avoid unnecessary `any` at boundaries? Compare `src/lib/types.ts:15-120` with `src/app/(main)/websites/WebsiteAddForm.tsx:17-17`.

## Doc 5: Debugging Guide

For a dashboard chart bug:

1. Confirm the component receives the right `websiteId` and date state (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-20`).
2. Confirm the query key contains the state you changed (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-23`).
3. Confirm the request URL and params are built by `useApi` (`src/components/hooks/useApi.ts:34-48`).
4. Confirm the route validates/authenticates successfully (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-28`).
5. Confirm `getQueryFilters` preserves your filter (`src/lib/request.ts:79-90`, `src/lib/request.ts:111-178`).
6. Confirm the selected DB path through `runQuery` (`src/lib/db.ts:22-36`).
7. Confirm the chart receives normalized series (`src/components/metrics/PageviewsChart.tsx:27-83`).

For a login bug, trace `src/app/api/auth/login/route.ts:12-48`, `src/lib/auth.ts:17-60`, and `src/components/hooks/useApi.ts:30-33`.

## Doc 6: Change Playbook

To add a feature end-to-end:

1. Find the closest existing page or component, such as `WebsiteChart` for dashboard chart changes (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:7-59`).
2. Find the matching query hook or create one using the same pattern (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`).
3. Add or update the API route schema and permission check (`src/app/api/websites/[websiteId]/pageviews/route.ts:12-28`).
4. Add query functions in `src/queries/sql` or `src/queries/prisma`, preserving `runQuery` when analytics storage matters (`src/queries/sql/pageviews/getPageviewStats.ts:9-102`).
5. Update types in `src/lib/types.ts` when the data shape or domain contract changes (`src/lib/types.ts:122-193`).
6. Add tests using Vitest for pure helpers or component behavior (`vitest.config.ts:10-14`) and Cypress for e2e flows (`package.json:44-45`).
7. Run `pnpm run test`, `pnpm run lint`, or targeted checks from `package.json:42-49`.

## Doc 7: Interview Walkthrough

Talk track:

"Umami is a privacy-focused analytics app. It uses Next.js App Router for UI and API routes, with a client provider layer for Zen UI, i18n, React Query, and error boundaries (`src/app/Providers.tsx:51-61`). Authenticated pages are wrapped by a main app shell that loads user/config/team state and redirects unauthenticated users (`src/app/(main)/App.tsx:13-92`). Dashboard components use typed React Query hooks to call API routes (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`). API routes validate input, check auth and permissions, then call query modules (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-33`). Analytics queries can run against PostgreSQL/Prisma or ClickHouse depending on environment (`src/lib/db.ts:22-36`). The core data model is Website, Session, WebsiteEvent, EventData, and related reporting entities (`prisma/schema.prisma:66-194`)."

Under pressure, explain the pageviews flow: `WebsiteChart` to hook to `useApi` to route to permissions to query dispatch to chart rendering (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/hooks/useApi.ts:20-74`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, `src/queries/sql/pageviews/getPageviewStats.ts:9-102`).

