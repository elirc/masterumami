# Mission Learning Path

This suite is a training campaign. Each mission asks you to trace real files, annotate real code, answer questions, and self-grade immediately.

Recommended pacing:

- Tier 1: one mission per sitting, 30 to 60 minutes each.
- Tier 2: one mission every one or two days, with a scratch branch open.
- Tier 3: one mission per week, ideally while reviewing a real or hypothetical change.

How to self-grade:

1. Read the exact file ranges.
2. Explain the flow without looking.
3. Compare your answer to the rubric.
4. Re-read the lines you could not explain.

This differs from normal documentation because you are not trying to memorize the codebase. You are training the skill of following ownership: UI owns interaction, hooks own fetching contracts, routes own validation and authorization, query modules own persistence, and schema owns the product nouns (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`, `src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, `prisma/schema.prisma:66-150`).

