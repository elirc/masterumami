# Journal

## Story Design Rationale

The story path starts with display-only work because low-risk changes build confidence in component anatomy. It then moves into existing data display, new API behavior, persisted preferences, schema-backed domain work, and finally caching/auth/performance architecture.

## Why These Stories Are Ordered This Way

Stories 1-3 use existing metric UI and chart code so the learner can practice safe edits before touching backend contracts (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:25-90`, `src/components/metrics/PageviewsChart.tsx:21-97`). Stories 4-5 add UI that reads existing website/date context (`src/components/hooks/useNavigation.ts:5-58`, `src/components/hooks/useDateRange.ts:8-37`). Stories 6-7 add new endpoint/state behavior using the existing route and storage patterns (`src/app/api/websites/[websiteId]/active/route.ts:6-25`, `src/app/(main)/App.tsx:19-32`). Stories 8-9 require schema/API/UI coordination around `Website` (`prisma/schema.prisma:66-97`, `src/app/api/websites/[websiteId]/route.ts:37-104`). Story 10 asks for a cache design that respects share-token auth and query-filter identity (`src/lib/auth.ts:42-48`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-32`).

## Code Paths Chosen

- `WebsiteMetricsBar` was chosen because it is user-visible, compact, and contains real edge-case math (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`).
- `WebsiteChart` and `PageviewsChart` were chosen because they show date-driven data display (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/metrics/PageviewsChart.tsx:27-97`).
- `src/app/api/websites/[websiteId]/route.ts` was chosen for hard stories because it already owns read/update/delete permissions and website persistence (`src/app/api/websites/[websiteId]/route.ts:16-125`).
- `src/app/api/share/[slug]/route.ts` and `src/lib/auth.ts` were chosen for the expert story because public share caching is only safe if share token boundaries stay intact (`src/app/api/share/[slug]/route.ts:47-105`, `src/lib/auth.ts:42-48`).

## What A Tech Lead Would Watch

A tech lead would keep early stories small, require explanations before merging medium stories, and demand explicit risk notes for hard stories. The main risks are broken query contracts (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-35`), skipped permissions (`src/permissions/website.ts:7-84`), schema drift (`prisma/schema.prisma:66-97`), and cache bleed between private and shared analytics (`src/components/hooks/useApi.ts:25-33`, `src/lib/auth.ts:42-48`).

