# Journal

## First-Pass Mental Model

Umami is a Next.js App Router analytics application: pages live under `src/app`, client-wide providers are installed in `src/app/Providers.tsx:36-62`, authenticated main navigation is wrapped by `src/app/(main)/App.tsx:13-92`, API handlers live under `src/app/api`, and analytics queries are dispatched to PostgreSQL/Prisma or ClickHouse through `src/lib/db.ts:22-36`.

The product domain is analytics. A `Website` is the tracked property (`prisma/schema.prisma:66-97`), a `Session` is a visitor context (`prisma/schema.prisma:34-64`), a `WebsiteEvent` is a pageview or event record (`prisma/schema.prisma:99-150`), and chart queries summarize those events through SQL query modules such as `src/queries/sql/pageviews/getPageviewStats.ts:9-102`.

## Important Discoveries

- Package management is pnpm because `pnpm-lock.yaml` exists and source installation says `pnpm install` (`README.md:33-39`).
- The dev command is `dotenv next dev --turbo`, while production build chains environment checks, DB client generation, migrations, tracker builds, recorder builds, geo build, and Next build (`package.json:13-35`).
- Runtime state is split: server data uses TanStack Query through `src/app/Providers.tsx:11-19` and `src/components/hooks/useApi.ts:44-74`, while app preferences and identity cache use Zustand in `src/store/app.ts:14-54`.
- Auth is bearer-token based for normal UI/API requests, with share-token support gated by share context headers (`src/lib/auth.ts:17-60`).
- The DB strategy is dual-path: ClickHouse is selected when `CLICKHOUSE_URL` exists, otherwise PostgreSQL/Prisma handles query modules (`src/lib/db.ts:22-36`).
- Seed data is first-class local tooling: `scripts/seed-data.ts:4-17` describes demo analytics data and `scripts/seed-data.ts:90-104` blocks production/cloud runs.

## Why These Teaching Anchors

- `src/app/layout.tsx:1-40` and `src/app/Providers.tsx:1-62` show where the UI receives fonts, metadata, routing, i18n, React Query, and error boundaries.
- `src/app/(main)/App.tsx:13-92` shows the authenticated shell, redirect behavior, team persistence, navigation, and optional self-tracking scripts.
- `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57` is compact but full-stack: it reads date state, calls a query hook, shapes comparison data, and renders a chart.
- `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70` is the best backend teaching route because it validates input, checks permissions, builds filters, runs parallel queries, handles comparison mode, and serializes JSON.
- `src/queries/sql/pageviews/getPageviewStats.ts:9-102` shows the core architecture decision: one public query function, two SQL implementations.

## Where A Junior Might Get Confused

- Next.js route groups like `src/app/(main)` are folders that affect organization but not the URL path; the main layout wraps authenticated pages at `src/app/(main)/layout.tsx:5-10`.
- The query hook `useWebsitePageviewsQuery` does not fetch directly with `fetch`; it gets a wrapped `get` function from `useApi` and delegates to TanStack Query (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:15-35`).
- Prisma models use mapped database names, so `WebsiteEvent.eventType` maps to `event_type` (`prisma/schema.prisma:99-150`).
- API handlers receive dynamic route params as promises in this Next version, as shown by `{ params }: { params: Promise<{ websiteId: string }> }` (`src/app/api/websites/[websiteId]/pageviews/route.ts:8-11`).

## Where A Mid-Level Engineer Should Slow Down

- Filter handling crosses URL params, Zod schemas, request parsing, segment/cohort expansion, and SQL fragment generation (`src/lib/request.ts:12-52`, `src/lib/request.ts:111-178`, `src/lib/prisma.ts:232-252`).
- Metrics can come from raw `website_event` rows or aggregated `website_event_stats_hourly` rows depending on ClickHouse filters and unit (`src/queries/sql/pageviews/getPageviewStats.ts:59-99`).
- `MetricCard` computes percent change from `value` and `change`, not from `previousValue`; this is valid for current callers but easy to misuse (`src/components/metrics/MetricCard.tsx:21-34`).

## Where A Senior Engineer Should Be Skeptical

- `useApi` dependency arrays include imported functions such as `httpGet`, but omit values like auth token, share headers, base path, and pathname-derived state from callback dependencies (`src/components/hooks/useApi.ts:20-74`).
- `useDateRange` memoizes using the whole `options` object, which can change identity every render when callers pass inline objects (`src/components/hooks/useDateRange.ts:8-24`).
- Raw SQL is parameterized through custom placeholder replacement in Prisma (`src/lib/prisma.ts:255-282`) and through ClickHouse query params (`src/lib/clickhouse.ts:265-287`), so review must verify every dynamic SQL segment is generated from controlled constants.
- `request` assumes every response body is JSON, so non-JSON error responses can become client-side parsing failures (`src/lib/fetch.ts:18-41`).

## How To Use Checkpoints

Do not treat checkpoint questions like trivia. Read the cited files, trace the data with your finger, answer the question, then self-grade. A strong answer names exact files, describes ownership boundaries, and predicts what would break if one piece changed.

