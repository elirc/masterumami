# Junior Engineer Guide

## Setup And Run

The source README requires Node.js 18.18+ and PostgreSQL v12.14+ (`README.md:28-31`). It instructs source installs to clone, enter the repo, and run `pnpm install` (`README.md:33-39`). It requires `.env` with `DATABASE_URL` in PostgreSQL URL form (`README.md:41-53`). The documented production path is `pnpm run build` then `pnpm run start` (`README.md:55-69`).

For local development, reconstruct the practical flow from scripts:

```bash
pnpm install
# create .env with DATABASE_URL=postgresql://username:password@localhost:5432/umami
pnpm run build-db
pnpm run update-db
pnpm run seed-data -- --days 30
pnpm run dev
```

Why those commands: `build-db` generates Prisma client and app client code (`package.json:26-31`), `update-db` deploys Prisma migrations (`package.json:33-34`), `seed-data` runs `scripts/seed-data.ts` (`package.json:46-46`), and `dev` starts Next through dotenv (`package.json:14-14`). Seed data is intended for local development and testing (`scripts/seed-data.ts:4-17`) and is blocked in production/cloud environments (`scripts/seed-data.ts:90-104`).

Docker setup exists for the released image and a bundled PostgreSQL service (`docker-compose.yml:1-35`). The compose file sets `DATABASE_URL` and `APP_SECRET` for the container (`docker-compose.yml:7-10`) and exposes port 3000 (`docker-compose.yml:5-6`).

## Folder Orientation

- `src/app`: Next.js App Router pages, layouts, and API routes. Root layout lives at `src/app/layout.tsx:1-40`; API handlers live under `src/app/api`, for example `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`.
- `src/components`: reusable UI, metric displays, hooks, inputs, charts, and SVG components. Query hooks live under `src/components/hooks/queries`, for example `src/components/hooks/queries/useWebsitePageviewsQuery.ts:1-36`.
- `src/lib`: cross-cutting helpers for auth, request parsing, database adapters, formatting, dates, fetch wrappers, constants, and types. Examples: `src/lib/auth.ts:17-60`, `src/lib/request.ts:12-52`, `src/lib/types.ts:15-31`.
- `src/queries`: database-facing read/write functions. Analytics SQL is under `src/queries/sql`, while ORM-style entities are under `src/queries/prisma` (`src/queries/sql/pageviews/getPageviewStats.ts:9-102`, `src/queries/prisma/website.ts:7-119`).
- `src/store`: Zustand stores for app preferences, cache, version, dashboard, and websites (`src/store/app.ts:14-54`).
- `prisma`: Prisma schema and migrations. The generated client output is configured in `prisma/schema.prisma:1-5`.
- `db`: database-specific SQL and ClickHouse migrations. ClickHouse analytics tables are referenced by query modules such as `src/queries/sql/pageviews/getPageviewStats.ts:80-99`.
- `cypress`: end-to-end tests, configured by scripts in `package.json:44-45`.

## One-Level Deeper: Frontend

1. `src/app/(main)`: authenticated application shell and product pages. `src/app/(main)/layout.tsx:5-10` wraps children in `App`, and `src/app/(main)/App.tsx:53-90` renders mobile/side/top nav plus children.
2. `src/components/hooks`: app-specific hooks for API, navigation, dates, filters, localization, and query helpers. `useNavigation` extracts ids from the path (`src/components/hooks/useNavigation.ts:5-14`) and returns router/query helpers (`src/components/hooks/useNavigation.ts:44-57`).
3. `src/components/metrics`: reusable chart and metric UI. `MetricCard` animates numbers and change labels (`src/components/metrics/MetricCard.tsx:21-72`), and `PageviewsChart` converts API data into Chart.js datasets (`src/components/metrics/PageviewsChart.tsx:21-97`).

## One-Level Deeper: Backend

1. `src/app/api`: route handlers. They typically define a Zod schema, call `parseRequest`, check permission, call query functions, and return `json` (`src/app/api/websites/[websiteId]/pageviews/route.ts:12-70`).
2. `src/lib`: backend middleware-like helpers. `parseRequest` handles query/body parsing, Zod validation, and auth (`src/lib/request.ts:12-52`); `checkAuth` parses bearer/share tokens and loads users (`src/lib/auth.ts:17-60`).
3. `src/queries`: persistence boundary. `getPageviewStats` dispatches to Prisma or ClickHouse (`src/queries/sql/pageviews/getPageviewStats.ts:9-13`); `createWebsite` wraps Prisma create (`src/queries/prisma/website.ts:113-119`).

## Frontend Entry Point Walkthrough

Source: `src/app/layout.tsx:1-40`.

```tsx
import type { Metadata } from 'next'; // Next metadata type for generateMetadata return shape.
import { headers } from 'next/headers'; // Reads request headers on the server.
import { Inter } from 'next/font/google'; // Loads the app font through Next.
import { Suspense } from 'react'; // Lets client providers/pages suspend while loading.
import { getBaseUrl } from '@/lib/get-base-url'; // Computes metadata base from request headers.
import { Providers } from './Providers'; // Client runtime wrapper for UI, router, i18n, query cache.

export default function ({ children }) {
  if (process.env.DISABLE_UI) {
    // Environment kill switch: render an empty shell when UI is disabled.
    return (
      <html>
        <body></body>
      </html>
    );
  }

  return (
    <html lang="en" className={`${inter.className} ${inter.variable}`}>
      <body>
        <Suspense>
          <Providers>{children}</Providers>
          {/* All app routes inherit client providers from here. */}
        </Suspense>
      </body>
    </html>
  );
}
```

## Backend Entry Point Walkthrough

Source: `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`.

```ts
export async function GET(request: Request, { params }: { params: Promise<{ websiteId: string }> }) {
  const schema = withDateRange({ ...filterParams });
  // Route accepts date params plus analytics filter params.

  const { auth, query, error } = await parseRequest(request, schema);
  // parseRequest validates query params and attaches auth.

  if (error) return error();
  // Validation or auth failure exits before touching data.

  const { websiteId } = await params;

  if (!(await canViewWebsite(auth, websiteId))) {
    return unauthorized();
  }
  // Domain permission check: only owners/team members/admin/share tokens can view.

  const filters = await getQueryFilters(query, websiteId);
  // Converts raw query params into DB-ready date/filter/cohort/segment options.

  const [pageviews, sessions] = await Promise.all([
    getPageviewStats(websiteId, filters),
    getSessionStats(websiteId, filters),
  ]);
  // Runs independent analytics summaries in parallel.

  return json({ pageviews, sessions });
}
```

## TypeScript Orientation

- Utility aliases turn constant objects into narrow unions, for example `ObjectValues<T>` and `Role` (`src/lib/types.ts:6-13`).
- API auth context is typed as optional user plus optional share token (`src/lib/types.ts:15-31`).
- Query filters are composed from date, filter, sort, paging, and segment interfaces (`src/lib/types.ts:66-120`).
- React Query option typing deliberately omits `queryKey` and `queryFn`, so feature hooks own their cache identity and fetch function (`src/lib/types.ts:1-9`).

## Three React Component Anatomies

### Simple Component: `MetricCard`

Source: `src/components/metrics/MetricCard.tsx:21-72`.

```tsx
export const MetricCard = ({
  value = 0,
  change = 0,
  label,
  formatValue = formatNumber,
  showChange = false,
}: MetricCardProps) => {
  const diff = value - change; // Caller passes the delta, so previous value is current minus delta.
  const pct = diff !== 0 ? ((value - diff) / diff) * 100 : value !== 0 ? 100 : 0;
  const props = useSpring({ x: Number(value) || 0, from: { x: 0 } }); // Animates the displayed value.

  return (
    <Column>
      <Text>{label}</Text>
      <AnimatedDiv>{props?.x?.to(x => formatValue(x))}</AnimatedDiv>
      {showChange && <ChangeLabel value={change}>{/* percent animation */}</ChangeLabel>}
    </Column>
  );
};
```

### Data Component: `WebsiteChart`

Source: `src/app/(main)/websites/[websiteId]/WebsiteChart.tsx:14-57`.

```tsx
const { timezone } = useTimezone(); // Reads app timezone preference.
const { dateRange, dateCompare } = useDateRange({ timezone }); // Converts URL/local settings into dates.
const { data, isLoading, isFetching, error } = useWebsitePageviewsQuery({
  websiteId,
  compare: compareMode ? dateCompare?.compare : undefined,
}); // Fetches chart data for the selected website and optional comparison range.

const chartData = useMemo(() => {
  if (!data) return { pageviews: [], sessions: [] };
  return { pageviews, sessions, ...(compare && { compare: {/* aligns comparison points */} }) };
}, [data, startDate, endDate, unit]);

return (
  <LoadingPanel data={data} isFetching={isFetching} isLoading={isLoading} error={error}>
    <PageviewsChart data={chartData} minDate={startDate} maxDate={endDate} unit={unit} />
  </LoadingPanel>
);
```

### Form Component: `WebsiteAddForm`

Source: `src/app/(main)/websites/WebsiteAddForm.tsx:14-59`.

```tsx
const { t, labels, messages } = useMessages(); // Localized labels and validation messages.
const { mutateAsync, error, isPending } = useUpdateQuery('/websites', { teamId });
// Mutation posts to /api/websites and includes teamId when creating under a team.

const handleSubmit = async (data: any) => {
  await mutateAsync(data, {
    onSuccess: async () => {
      onSave?.(); // Parent can refresh/close after success.
      onClose?.();
    },
  });
};

return (
  <Form onSubmit={handleSubmit} error={error?.message}>
    <FormField name="name" rules={{ required: t(labels.required) }}>
      <TextField autoComplete="off" />
    </FormField>
    <FormField name="domain" rules={{ pattern: { value: DOMAIN_REGEX, message: t(messages.invalidDomain) } }}>
      <TextField autoComplete="off" />
    </FormField>
  </Form>
);
```

## Three Backend Route Anatomies

### Analytics Read Route

Source: `src/app/api/websites/[websiteId]/pageviews/route.ts:12-70`.

```ts
const schema = withDateRange({ ...filterParams }); // Accepts date and filter query params.
const { auth, query, error } = await parseRequest(request, schema); // Validates and authenticates.
if (error) return error();
const { websiteId } = await params; // Dynamic route segment.
if (!(await canViewWebsite(auth, websiteId))) return unauthorized(); // Authorization.
const filters = await getQueryFilters(query, websiteId); // DB-ready filters.
const [pageviews, sessions] = await Promise.all([getPageviewStats(websiteId, filters), getSessionStats(websiteId, filters)]);
return json({ pageviews, sessions }); // Stable response shape for chart hook.
```

### Website Collection Route

Source: `src/app/api/websites/route.ts:14-36`.

```ts
const schema = z.object({ ...pagingParams, ...searchParams, includeTeams: z.string().optional() });
const { auth, query, error } = await parseRequest(request, schema);
if (error) return error();
const filters = await getQueryFilters(query);
return query.includeTeams
  ? json(await getAllUserWebsitesIncludingTeamOwner(auth.user.id, filters))
  : json(await getUserWebsites(auth.user.id, filters));
```

### Login Route

Source: `src/app/api/auth/login/route.ts:12-48`.

```ts
const schema = z.object({ username: z.string(), password: z.string() });
const { body, error } = await parseRequest(request, schema, { skipAuth: true }); // Login cannot require auth.
if (error) return error();
const user = await getUserByUsername(body.username, { includePassword: true });
if (!user || !checkPassword(body.password, user.password)) return unauthorized({ code: 'incorrect-username-password' });
const token = redis.enabled ? await saveAuth({ userId: user.id, role: user.role }) : createSecureToken({ userId: user.id, role: user.role }, secret());
return json({ token, user: { id: user.id, username: user.username, role: user.role, isAdmin: user.role === ROLES.admin, teams } });
```

## Domain Glossary

- Website: a tracked property with name, domain, owner/team, replay settings, and soft-delete/reset metadata (`prisma/schema.prisma:66-97`).
- Session: a visitor identity/context with browser, OS, device, geography, language, and distinct id (`prisma/schema.prisma:34-64`).
- WebsiteEvent: an analytics event or pageview tied to website/session/visit with URL, referrer, UTM, event name, tag, hostname, and performance fields (`prisma/schema.prisma:99-150`).
- Visit: a grouping id on `WebsiteEvent` used alongside session for visit-level metrics (`prisma/schema.prisma:100-104`).
- EventData: custom event properties stored by key/type/value (`prisma/schema.prisma:152-171`).
- Segment: saved filters for a website report/dashboard view (`prisma/schema.prisma:253-265`).
- Cohort: a filter mode expanded by `getQueryFilters` using segment parameters and date range (`src/lib/request.ts:134-160`).
- Share token: a token that can view shared entities only when share context headers are present (`src/lib/auth.ts:42-48`).
- Query hook: a React hook that owns a query key and fetch function, for example `useWebsitePageviewsQuery` (`src/components/hooks/queries/useWebsitePageviewsQuery.ts:11-35`).

## Junior Socratic Checkpoint

1. What file installs React Query for the whole client app?
2. What route serves website pageview chart data?
3. Where does `websiteId` enter the API route?
4. Where are date range params converted into actual `Date` objects?
5. What model stores pageview URL paths?
6. Which file decides whether analytics queries use ClickHouse or Prisma?
7. Why does a website creation route need both schema validation and permission checks?

## How To Self-Grade

A strong answer names `src/app/Providers.tsx:11-19`, `src/app/api/websites/[websiteId]/pageviews/route.ts:8-70`, `src/lib/request.ts:63-77`, `prisma/schema.prisma:99-150`, `src/lib/db.ts:22-36`, and `src/app/api/websites/route.ts:38-99`. It should also explain ownership: hooks fetch, routes authorize, query modules read/write data, and schema defines persistence.

