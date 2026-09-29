# Architectural Cartographer

This suite is a top-down map of the Umami codebase for a product owner growing into a hands-on maintainer. It teaches how the system is shaped, where to start reading, how UI routes connect to API routes, how analytics data reaches charts, and where a reviewer should slow down.

Use it in this order:

1. `00-reading-map.md` gives the mental model and first files to open.
2. `01-junior-engineer.md` teaches setup, folders, entry points, components, routes, and vocabulary.
3. `02-mid-level-engineer.md` traces architecture, state, contracts, and one end-to-end feature.
4. `03-senior-engineer.md` critiques design, security, performance, tests, and ownership risks.
5. `04-reference-suite.md` is the durable reference set for onboarding, review, debugging, change work, and interviews.

Every codebase-specific claim cites source lines, for example `src/app/Providers.tsx:11-19`. When you hit a Socratic checkpoint, answer aloud or in a scratch note, then compare against the included self-grading rubric immediately.

Teaching anchors used throughout:

- App shell and providers: `src/app/layout.tsx:1-40`, `src/app/Providers.tsx:1-62`, `src/app/(main)/App.tsx:13-92`.
- Analytics chart flow: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, `src/queries/sql/pageviews/getPageviewStats.ts:9-102`.
- Auth and permissions: `src/lib/auth.ts:17-60`, `src/permissions/website.ts:7-40`.
- Persistence: `prisma/schema.prisma:66-150`, `src/lib/db.ts:22-36`, `src/lib/prisma.ts:255-282`, `src/lib/clickhouse.ts:265-287`.

