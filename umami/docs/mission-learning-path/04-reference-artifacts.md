# Mission 25: Write the Docs That Don't Exist

## Doc 1: Junior Onboarding Checklist

- Install dependencies with `pnpm install` after reading source install instructions (`README.md:33-39`).
- Create `.env` with `DATABASE_URL` (`README.md:41-53`).
- Learn scripts in `package.json:13-49`, especially `dev`, `build`, `update-db`, `seed-data`, `test`, and `lint`.
- Read root layout and providers (`src/app/layout.tsx:1-40`, `src/app/Providers.tsx:1-62`).
- Trace one feature: pageviews chart from UI to query (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`).
- First PR checklist: schema or prop typed, route validates input, permission check exists, query key includes all data-affecting params, test or manual verification named.

## Doc 2: Architecture Guide For New Engineers

Navigate by ownership:

- App shell: `src/app/layout.tsx:1-40`, `src/app/(main)/App.tsx:13-92`.
- Client runtime: `src/app/Providers.tsx:11-62`.
- API transport: `src/components/hooks/useApi.ts:20-74`.
- Request parsing: `src/lib/request.ts:12-52`.
- Permissions: `src/permissions/website.ts:7-40`.
- Analytics queries: `src/queries/sql/pageviews/getPageviewStats.ts:9-102`.
- Data model: `prisma/schema.prisma:66-194`.

## Doc 3: Code Review Checklist

- Does the hook own a complete query key? Compare `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-35`.
- Does the route call `parseRequest` and handle `error`? Compare `src/app/api/websites/[websiteId]/pageviews/route.ts:16-20`.
- Does authorization happen before query calls? Compare `src/app/api/websites/[websiteId]/pageviews/route.ts:22-33`.
- Does new analytics code maintain Prisma and ClickHouse behavior? Compare `src/queries/sql/pageviews/getPageviewStats.ts:16-102`.
- Does UI math handle empty data? Inspect `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`.
- Does raw SQL use parameter helpers? Inspect `src/lib/prisma.ts:255-282` and `src/lib/clickhouse.ts:265-287`.

## Doc 4: Debugging Playbook

Dashboard data missing:

1. Confirm component receives id/date: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-20`.
2. Confirm query key/fetch params: `src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-32`.
3. Confirm auth headers: `src/components/hooks/useApi.ts:25-33`.
4. Confirm route validation/permission: `src/app/api/websites/[websiteId]/pageviews/route.ts:16-28`.
5. Confirm filter expansion: `src/lib/request.ts:111-178`.
6. Confirm DB branch: `src/lib/db.ts:22-36`.

Login failing:

1. Check login body schema: `src/app/api/auth/login/route.ts:12-18`.
2. Check password verification: `src/app/api/auth/login/route.ts:26-30`.
3. Check token creation: `src/app/api/auth/login/route.ts:34-47`.
4. Check auth parsing: `src/lib/auth.ts:17-60`.

## Doc 5: Change Playbook

1. Branch.
2. Find nearest existing pattern.
3. Add UI in `src/app` or `src/components`.
4. Add hook in `src/components/hooks/queries` if server data is needed.
5. Add API route in `src/app/api` with Zod schema, `parseRequest`, and permission check.
6. Add query function in `src/queries/prisma` for entity CRUD or `src/queries/sql` for analytics.
7. Update `prisma/schema.prisma` only when the domain model changes.
8. Add or update tests. Vitest looks for `src/**/*.test.{ts,tsx}` (`vitest.config.ts:10-14`).
9. Run relevant scripts from `package.json:42-49`.

## Doc 6: Senior Ownership Notes

Monitor:

- Query latency around ClickHouse rollup/raw branches (`src/queries/sql/pageviews/getPageviewStats.ts:59-99`).
- Auth and share token boundaries (`src/lib/auth.ts:42-48`).
- Setup drift between README and scripts (`README.md:28-69`, `package.json:13-49`).
- Seed data usefulness for local debugging (`scripts/seed-data.ts:4-17`).

Improve:

- Replace risky `any` at API/component boundaries (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:21-46`).
- Guard empty metrics (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`).
- Add parity tests for analytics query paths (`src/queries/sql/pageviews/getPageviewStats.ts:16-102`).

Leave alone unless needed:

- Provider stack shape, because it cleanly centralizes runtime wrappers (`src/app/Providers.tsx:51-61`).
- Permission helper pattern, because it keeps routes readable (`src/permissions/website.ts:7-152`).

## Doc 7: Interview Walkthrough

Practice answer:

"Umami is a Next.js App Router analytics app. The root layout sets document structure and wraps pages in client providers (`src/app/layout.tsx:14-37`, `src/app/Providers.tsx:51-61`). Authenticated pages pass through a main shell that loads login/config/team state and redirects on auth errors (`src/app/(main)/App.tsx:13-43`). Dashboard components use custom query hooks to call `/api` routes through a shared `useApi` wrapper (`src/components/hooks/useApi.ts:20-74`). Routes validate with Zod through `parseRequest`, authorize with domain permission helpers, then call query modules (`src/lib/request.ts:12-52`, `src/permissions/website.ts:7-40`). Analytics query functions choose ClickHouse or Prisma/PostgreSQL through `runQuery` (`src/lib/db.ts:22-36`). The core data model is Website, Session, WebsiteEvent, EventData, Segment, Report, and related entities (`prisma/schema.prisma:66-285`)."

