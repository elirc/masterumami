# User Stories

## Story 1: Rename A Metric Label In The Website Dashboard
**Difficulty:** Easy
**Estimated Time:** 1 hour
**Skills You'll Practice:** component reading, localization awareness, safe UI changes
**The Story:** As a product owner, I want the website metric label "views" to be displayed as "page views" in the dashboard so that the analytics meaning is clearer.
**Acceptance Criteria:**
- [ ] The dashboard metric card currently using `labels.views` in the website metrics bar displays the new wording.
- [ ] The pageviews chart legend currently using `labels.views` displays matching wording.
- [ ] No API, database, or query code is changed.
**Files You'll Likely Touch:**
- `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx` - the website metrics bar renders views with `t(labels.views)` (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:40-43`).
- `src/components/metrics/PageviewsChart.tsx` - chart datasets label the views series with `t(labels.views)` (`src/components/metrics/PageviewsChart.tsx:43-52`).
- `public/intl/messages/en-US.json` - English messages include both `page-views` and `views` keys (`public/intl/messages/en-US.json:216-216`, `public/intl/messages/en-US.json:370-371`).
**High-Level Implementation Plan:**
1. Find the message key behind `labels.views` by searching `views` in `public/intl/messages/en-US.json`.
2. Change the English value to "Page views".
3. Run the UI and verify `WebsiteMetricsBar` and `PageviewsChart` now show the same wording.
**Tips:**
- Do not change the metric object field `pageviews`; backend responses use that name (`src/app/api/websites/[websiteId]/pageviews/route.ts:55-69`).
- Keep the change at the display-message layer because the domain field still means pageview counts (`src/queries/sql/pageviews/getPageviewStats.ts:27-40`).
- Use `rg "labels.views|\"views\""` to find all display uses.
**What Could Go Wrong:**
- Changing a data field name instead of a message breaks chart data.
- Updating only one locale leaves other languages unchanged.
- Changing `pageviews` to `pageViews` in API code breaks the hook contract (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:6-13`).
**Stretch Goal:** Add a short glossary note in an internal docs page explaining "views" versus "sessions" using schema fields (`prisma/schema.prisma:34-150`).
**Connects To:** Story 2, because both start with safe display-only changes.

## Story 2: Add A Tooltip To Visit Duration
**Difficulty:** Easy
**Estimated Time:** 1.5 hours
**Skills You'll Practice:** props, reusable components, metric UX
**The Story:** As a dashboard user, I want a tooltip on visit duration so that I understand how the number is calculated.
**Acceptance Criteria:**
- [ ] The visit duration metric card shows an info tooltip.
- [ ] Tooltip text explains that duration is total tracked time divided by visits.
- [ ] Existing number formatting and comparison behavior remain unchanged.
**Files You'll Likely Touch:**
- `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx` - visit duration metric is defined with `totaltime / visits` and `formatShortTime` (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:55-62`).
- `src/components/metrics/MetricCard.tsx` - component already accepts `tooltip` and renders a tooltip button (`src/components/metrics/MetricCard.tsx:9-19`, `src/components/metrics/MetricCard.tsx:51-60`).
**High-Level Implementation Plan:**
1. Add a `tooltip` field to the visit duration metric object in `WebsiteMetricsBar`.
2. Pass `tooltip` through the `metrics.map` destructuring and into `MetricCard`.
3. Verify `MetricCard` renders the existing tooltip UI.
**Tips:**
- `MetricCard` already owns tooltip rendering; reuse it instead of adding custom tooltip JSX in the parent (`src/components/metrics/MetricCard.tsx:51-60`).
- Keep the metric math unchanged for this story (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:55-62`).
- Use localized message helpers if nearby code already uses `t`, `labels`, and `messages` (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:16-17`).
**What Could Go Wrong:**
- Forgetting to pass `tooltip` through the map means the prop is never used.
- Putting a button inside another button-like element could hurt accessibility.
- Editing `MetricCard` in a way that affects all metric cards globally.
**Stretch Goal:** Add a tooltip to bounce rate that explains bounces use single-page visits.
**Connects To:** Story 3, because you will continue shaping metric UI from existing data.

## Story 3: Hide Broken Empty-Site Metric Math
**Difficulty:** Easy
**Estimated Time:** 2 hours
**Skills You'll Practice:** edge cases, defensive rendering, small refactor
**The Story:** As a new Umami user, I want an empty website dashboard to show clean zero values so that I do not see `NaN` or `Infinity`.
**Acceptance Criteria:**
- [ ] Bounce rate displays `0%` when `visits` is zero.
- [ ] Visit duration displays `0s` or the existing zero-time format when `visits` is zero.
- [ ] Comparison math does not divide by zero when `comparison.visits` is zero.
**Files You'll Likely Touch:**
- `src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx` - current bounce rate and duration divide by `visits` and `comparison.visits` (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:45-61`).
**High-Level Implementation Plan:**
1. In `WebsiteMetricsBar`, compute safe local variables for current and previous bounce rate.
2. Compute safe local variables for current and previous visit duration.
3. Replace inline divisions in metric objects with those variables.
4. Manually test with mocked or local empty data.
**Tips:**
- Keep the `MetricCard` API unchanged because this is a parent calculation issue (`src/components/metrics/MetricCard.tsx:9-30`).
- Preserve `reverseColors: true` for bounce rate (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:52-53`).
- Preserve `formatShortTime` for duration formatting (`src/app/(main)/websites/[websiteId]/WebsiteMetricsBar.tsx:56-61`).
**What Could Go Wrong:**
- Fixing only current values but not comparison values still leaves `NaN` in change calculation.
- Changing the raw API response masks the UI edge case in the wrong layer.
- Returning `null` instead of `0` may break formatters expecting numbers.
**Stretch Goal:** Add a Vitest or component test for zero visits. Vitest includes `src/**/*.test.{ts,tsx}` (`vitest.config.ts:10-14`).
**Connects To:** Story 4, because you will read existing data more confidently after guarding edge cases.

## Story 4: Add A Compact Website ID Copy Field
**Difficulty:** Medium
**Estimated Time:** 3 hours
**Skills You'll Practice:** existing data display, clipboard UI, settings page orientation
**The Story:** As a site owner, I want to copy a website's tracking ID from the settings screen so that I can install the tracking script without hunting through URLs.
**Acceptance Criteria:**
- [ ] Website settings page shows the current website id in a compact read-only field.
- [ ] A copy action copies the id to clipboard.
- [ ] The feature uses existing website data already loaded for the settings page.
**Files You'll Likely Touch:**
- `src/app/(main)/settings/websites/[websiteId]/WebsiteSettingsPage.tsx` - settings page for a website exists in the App Router tree.
- `src/components/hooks/useNavigation.ts` - extracts `websiteId` from `/websites/:id` style URLs (`src/components/hooks/useNavigation.ts:9-14`).
- `src/components/input` components - existing input/button patterns live under this folder according to repository structure.
**High-Level Implementation Plan:**
1. Open `WebsiteSettingsPage.tsx` and identify how website data is loaded.
2. Add a read-only text field or compact row showing the website id.
3. Add a copy button using `navigator.clipboard.writeText(websiteId)`.
4. Verify the page works for a real website id route.
**Tips:**
- Use existing UI primitives from `@umami/react-zen`, matching files like `WebsiteAddForm` (`src/app/(main)/websites/WebsiteAddForm.tsx:1-4`).
- Get `websiteId` from page props or `useNavigation`, whichever the settings page already uses.
- Do not add a new API route; website id is already part of the route and entity identity.
**What Could Go Wrong:**
- Clipboard API may fail on non-secure contexts; show a safe visual fallback if needed.
- Adding the field in the wrong settings page could affect all websites rather than one website.
- Accidentally exposing more fields than the user is authorized to view.
**Stretch Goal:** Add copy feedback using the existing toast pattern from `useUpdateQuery` (`src/components/hooks/queries/useUpdateQuery.ts:11-14`).
**Connects To:** Story 5, because both add UI that reads existing state instead of creating new backend behavior.

## Story 5: Show Active Date Range Summary Above The Chart
**Difficulty:** Medium
**Estimated Time:** 3 hours
**Skills You'll Practice:** hooks, date state, presentational UI
**The Story:** As an analyst, I want the chart area to show the active date range summary so that screenshots have context.
**Acceptance Criteria:**
- [ ] Website chart area shows start and end date for the current date range.
- [ ] Summary updates when date range changes.
- [ ] Existing chart data fetching and chart rendering remain unchanged.
**Files You'll Likely Touch:**
- `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx` - already reads `dateRange` from `useDateRange` (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-16`).
- `src/components/hooks/useDateRange.ts` - returns `dateRange` with `startDate` and `endDate` (`src/components/hooks/useDateRange.ts:26-37`).
- `src/lib/format.ts` - likely date/number formatting helpers live here, with tests nearby in `src/lib/format.test.ts` according to repository structure.
**High-Level Implementation Plan:**
1. In `WebsiteChart`, derive a display string from `startDate` and `endDate`.
2. Render it near the chart title or above `PageviewsChart`.
3. Keep `useWebsitePageviewsQuery` unchanged.
4. Verify changing date params updates the summary.
**Tips:**
- `WebsiteChart` already destructures `startDate` and `endDate` (`src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:16-16`).
- Avoid putting display-only text into the query hook; hooks should fetch data, not decide layout (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`).
- Use locale-aware helpers if available through `useLocale` or existing formatting patterns.
**What Could Go Wrong:**
- Formatting dates without timezone awareness may confuse users in non-UTC zones.
- Adding `dateRange` object directly to memo dependencies can cause extra recompute if object identity changes.
- Overlapping the chart if the layout is too cramped.
**Stretch Goal:** Include comparison date range when `compareMode` is enabled.
**Connects To:** Story 6, because you will move from display-only date state to a new server-backed summary.

## Story 6: Add A Website Health Endpoint
**Difficulty:** Medium
**Estimated Time:** 5 hours
**Skills You'll Practice:** API route, auth, permission, lightweight backend response
**The Story:** As a site owner, I want to know whether a website has received any recent traffic so that I can confirm tracking is installed.
**Acceptance Criteria:**
- [ ] New endpoint returns whether the website has at least one event in the last 24 hours.
- [ ] Endpoint validates auth and uses `canViewWebsite`.
- [ ] UI can display "Receiving traffic" or "No recent traffic" from the endpoint.
**Files You'll Likely Touch:**
- `src/app/api/websites/[websiteId]/active/route.ts` - existing active endpoint pattern parses auth, checks `canViewWebsite`, calls `getActiveVisitors`, and returns JSON (`src/app/api/websites/[websiteId]/active/route.ts:6-25`).
- `src/app/api/websites/[websiteId]/pageviews/route.ts` - reference route validates and authorizes website access (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-28`).
- `src/permissions/website.ts` - `canViewWebsite` authorization logic (`src/permissions/website.ts:7-40`).
- `src/queries/sql` - analytics query modules for recent event checks.
**High-Level Implementation Plan:**
1. Create or extend an API route under `src/app/api/websites/[websiteId]/health/route.ts`.
2. Follow the `parseRequest` and `canViewWebsite` pattern from pageviews route.
3. Add a query function that checks recent `WebsiteEvent` activity for the website.
4. Add a query hook under `src/components/hooks/queries`.
5. Display status on the website page or settings page.
**Tips:**
- Use `parseRequest` even if the endpoint has no body, because it attaches auth (`src/lib/request.ts:12-52`).
- Keep response shape tiny, for example `{ active: boolean, since: string }`.
- Decide whether the query belongs in Prisma or SQL based on whether it reads analytics event tables (`prisma/schema.prisma:99-150`).
**What Could Go Wrong:**
- Querying without permission leaks whether a private website has traffic.
- Using local time instead of UTC can make "last 24 hours" inconsistent.
- Only implementing PostgreSQL when local dev or production path expects ClickHouse.
**Stretch Goal:** Show the timestamp of the most recent event.
**Connects To:** Story 7, because both add state or API behavior used by UI.

## Story 7: Remember The Last Open Website Tab
**Difficulty:** Medium
**Estimated Time:** 5 hours
**Skills You'll Practice:** local state, navigation, persistence, UX continuity
**The Story:** As a returning analyst, I want Umami to remember the last section I viewed for a website so that I can resume my work quickly.
**Acceptance Criteria:**
- [ ] When a user opens a website subpage, the app stores that section for the current website.
- [ ] Returning to the website can route or suggest the last section.
- [ ] Stored state is scoped per website id.
**Files You'll Likely Touch:**
- `src/components/hooks/useNavigation.ts` - extracts `websiteId` and path (`src/components/hooks/useNavigation.ts:5-14`, `src/components/hooks/useNavigation.ts:44-57`).
- `src/lib/storage.ts` - storage helpers are used by app state and navigation effects (`src/app/(main)/App.tsx:9-10`, `src/app/(main)/App.tsx:19-25`).
- `src/app/(main)/websites/[websiteId]/layout.tsx` - website-level layout fetches the website, guards deleted/missing websites, and wraps children in `WebsiteLayout` (`src/app/(main)/websites/[websiteId]/layout.tsx:5-27`).
- `src/components/hooks/useWebsiteNavItems.tsx` - website nav items define section ids and paths (`src/components/hooks/useWebsiteNavItems.tsx:16-29`, `src/components/hooks/useWebsiteNavItems.tsx:29-158`).
- `src/store/app.ts` or a new store file - Zustand store pattern uses `create` and setter functions (`src/store/app.ts:1-54`).
**High-Level Implementation Plan:**
1. Identify website section navigation in the website layout or nav-item hook.
2. Store `{ websiteId: sectionPath }` in local storage when path changes.
3. On website landing page, read the last section and route or show a button.
4. Add guard so missing/invalid sections fall back to the default overview.
**Tips:**
- Follow the `LAST_TEAM_CONFIG` persistence pattern in `App` (`src/app/(main)/App.tsx:19-25`).
- Use `useNavigation` instead of reparsing `window.location` manually (`src/components/hooks/useNavigation.ts:5-58`).
- Keep this client-side; no backend or database is required for local preference.
**What Could Go Wrong:**
- Storing one global section sends every website to the same tab.
- Redirecting too early can loop if the current path already equals the stored path.
- Persisting deleted routes can lead to 404s.
**Stretch Goal:** Add a preferences toggle to disable this behavior.
**Connects To:** Story 8, because you will need navigation and backend changes together next.

## Story 8: Add Notes To A Website
**Difficulty:** Hard
**Estimated Time:** 10 hours
**Skills You'll Practice:** schema change, Prisma migration, API CRUD, settings UI
**The Story:** As a site owner, I want to save internal notes on a website so that my team remembers implementation details and business context.
**Acceptance Criteria:**
- [ ] Website can store an optional notes field.
- [ ] Authorized users can view notes on website settings.
- [ ] Users with update permission can edit notes.
- [ ] Notes are persisted and survive reload.
**Files You'll Likely Touch:**
- `prisma/schema.prisma` - `Website` model currently has name, domain, ownership, timestamps, replay settings, and relations (`prisma/schema.prisma:66-97`).
- `src/queries/prisma/website.ts` - website read/update functions live here (`src/queries/prisma/website.ts:7-23`, `src/queries/prisma/website.ts:121-130`).
- `src/app/api/websites/[websiteId]/route.ts` - individual website route owns read, update, and delete handlers with view/update/delete permission checks (`src/app/api/websites/[websiteId]/route.ts:16-35`, `src/app/api/websites/[websiteId]/route.ts:37-104`, `src/app/api/websites/[websiteId]/route.ts:106-125`).
- `src/permissions/website.ts` - view/update permission helpers (`src/permissions/website.ts:7-84`).
- `src/app/(main)/settings/websites/[websiteId]/WebsiteSettingsPage.tsx` - settings UI.
**High-Level Implementation Plan:**
1. Add nullable `notes` field to `Website` in Prisma schema.
2. Generate and apply a migration.
3. Update Prisma query/update path to include notes.
4. Update API route schema to accept notes on update.
5. Add a settings form control for notes.
6. Confirm view and update permissions use existing helpers.
**Tips:**
- Keep notes on `Website`, not `WebsiteEvent`; notes describe the tracked property, not each event (`prisma/schema.prisma:66-150`).
- Use `canUpdateWebsite` for edits, not just `canViewWebsite` (`src/permissions/website.ts:58-84`).
- Follow existing Zod body schema style from `POST /api/websites` (`src/app/api/websites/route.ts:38-47`).
**What Could Go Wrong:**
- Forgetting migration means local schema and database drift.
- Allowing any viewer to edit notes breaks team permissions.
- A too-small column length truncates useful notes.
**Stretch Goal:** Show notes in the website list as a small icon or preview.
**Connects To:** Story 9, because both require schema/API/UI coordination.

## Story 9: Add A Saved Date Preset Per Website
**Difficulty:** Hard
**Estimated Time:** 12 hours
**Skills You'll Practice:** domain modeling, settings UX, date hooks, API persistence
**The Story:** As an analyst, I want each website to remember a preferred default date range so that different sites open with the most useful reporting window.
**Acceptance Criteria:**
- [ ] Website settings allow choosing a default date range preset.
- [ ] Website dashboard uses that preset when no explicit date query param exists.
- [ ] Explicit URL date params still override the website default.
- [ ] Preference is persisted per website.
**Files You'll Likely Touch:**
- `prisma/schema.prisma` - website model can store per-website settings (`prisma/schema.prisma:66-97`).
- `src/components/hooks/useDateRange.ts` - date range currently reads URL query, local storage, and default value (`src/components/hooks/useDateRange.ts:8-24`).
- `src/lib/constants.ts` - date range config and defaults are imported by date range hook (`src/components/hooks/useDateRange.ts:4-5`).
- `src/app/api/websites/[websiteId]/route.ts` - website update endpoint validates a body schema and calls `canUpdateWebsite` before `updateWebsite` (`src/app/api/websites/[websiteId]/route.ts:37-76`).
- `src/app/(main)/settings/websites/[websiteId]/WebsiteSettingsPage.tsx` - settings UI.
**High-Level Implementation Plan:**
1. Add a nullable `defaultDateRange` field or JSON setting to `Website`.
2. Add update support in website API schema.
3. Add settings UI to select a preset.
4. Update `useDateRange` or a website page wrapper so website default is used only when URL date is absent.
5. Preserve existing local/global fallback behavior.
**Tips:**
- Current fallback order in `useDateRange` is URL date, local storage, default constant (`src/components/hooks/useDateRange.ts:13-24`).
- Avoid making the hook fetch website data by itself if that creates circular data dependencies.
- URL params should win because shared dashboard links need deterministic views (`src/components/hooks/useNavigation.ts:14-57`).
**What Could Go Wrong:**
- Website default overriding explicit URL params makes shared links misleading.
- Fetching website settings too late can cause date range flicker.
- Storing arbitrary strings without validation can break `parseDateRange`.
**Stretch Goal:** Add team-level default date range inheritance.
**Connects To:** Story 10, because both involve architecture decisions around caching, defaults, and data ownership.

## Story 10: Add Cached Public Share Snapshots
**Difficulty:** Expert
**Estimated Time:** 20 hours
**Skills You'll Practice:** architecture, caching, auth boundaries, performance, invalidation
**The Story:** As a public share viewer, I want shared dashboards to load quickly even during traffic spikes so that public analytics links remain responsive.
**Acceptance Criteria:**
- [ ] Public share chart responses can be cached for a short TTL.
- [ ] Cache key includes share token/entity, website id, date range, filters, and compare mode.
- [ ] Authenticated private dashboard responses are not accidentally served from public cache.
- [ ] Cache invalidates or expires safely when share settings change.
- [ ] Implementation documents cache behavior and failure mode.
**Files You'll Likely Touch:**
- `src/lib/auth.ts` - share tokens require share context header and are distinct from user auth (`src/lib/auth.ts:42-48`).
- `src/components/hooks/useApi.ts` - share paths attach share token and share context headers (`src/components/hooks/useApi.ts:21-33`).
- `src/app/api/share/[slug]/route.ts` - share route loads a share by slug, resolves its entity, creates token data, and returns JSON (`src/app/api/share/[slug]/route.ts:47-105`).
- `src/app/api/websites/[websiteId]/pageviews/route.ts` - shared pageviews still go through website route and permission checks (`src/app/api/websites/[websiteId]/pageviews/route.ts:16-28`).
- `src/lib/redis.ts` - Redis helper supports TTL-backed `set`, `get`, `fetch`, and `remove`, and is enabled by `REDIS_URL` (`src/lib/redis.ts:34-52`, `src/lib/redis.ts:84-107`, `src/lib/redis.ts:151-153`).
- `src/lib/request.ts` - query filters define cache-relevant params (`src/lib/request.ts:111-178`).
**High-Level Implementation Plan:**
1. Trace public share loading from share page to API calls.
2. Define a cache helper for share-only analytics responses.
3. Build cache key from share identity and all query-affecting params.
4. Wrap pageviews route response for share-context requests only.
5. Add TTL and safe fallback to live query when cache fails.
6. Invalidate or naturally expire when share settings change.
7. Add tests for cache-key separation between share and authenticated requests.
**Tips:**
- Never use only `websiteId` as a cache key; filters and date range change the answer (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:19-32`).
- Share tokens are not normal auth tokens; preserve the `SHARE_CONTEXT_HEADER` rule (`src/lib/auth.ts:42-48`).
- Keep private dashboard cache separate because normal auth headers are bearer tokens (`src/components/hooks/useApi.ts:30-33`).
- Redis may not be enabled in every install, so design a no-cache fallback similar to auth's Redis conditional (`src/lib/auth.ts:62-74`).
**What Could Go Wrong:**
- Cache bleed could expose one shared dashboard's data through another share link.
- Missing a filter in the key returns wrong analytics for filtered charts.
- Long TTL makes public shares stale after settings or access changes.
**Stretch Goal:** Add lightweight cache metrics for hit/miss counts in development logs.
**Connects To:** This is the capstone story. It uses routing, hooks, auth, filters, query performance, and operational judgment from all previous stories.
