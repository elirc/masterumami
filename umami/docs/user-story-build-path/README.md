# User Story Build Path

This suite gives you ten real feature tickets to build in this codebase. The point is not to copy code from an answer. The point is to touch real Umami patterns: App Router pages, React components, query hooks, route handlers, permission helpers, Prisma/SQL query modules, and tests.

Difficulty progression:

- Stories 1-3: Easy. Small UI changes in one or two files.
- Stories 4-5: Medium-Easy. New UI that reads existing data through existing hooks/routes.
- Stories 6-7: Medium. New API endpoint or new state path.
- Stories 8-9: Hard. Full-stack features touching UI, backend, and persistence.
- Story 10: Expert. Architectural feature with caching/performance/auth implications.

A story is done when every acceptance criterion is independently verifiable, the implementation follows nearby patterns, and you can explain the changed files out loud. Use the app's existing route/hook/API/query patterns as your compass: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, and `src/queries/sql/pageviews/getPageviewStats.ts:9-102`.

How to get unstuck without being handed the answer:

```text
I'm working on Story X. I'm stuck on Y. Here is what I've tried: Z.
Don't give me the solution. Ask me questions that help me figure it out.
Use these code paths as context: [paste file paths and line ranges].
```

