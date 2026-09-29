# Domain Glossary

Analytics has its own nouns, and Umami overloads a few (a "visit" is not a "session"; a "website" is sometimes any tracked "entity"). This glossary pins each term to code so you stop guessing. Confusing pairs are called out explicitly.

## Core analytics nouns

| Term | Meaning in Umami | Code anchor |
| --- | --- | --- |
| **Event** | Any tracked action: a pageview, custom event, link/pixel hit, or performance sample. Stored one row per event. | `WebsiteEvent` model, `schema.prisma:99`; `EVENT_TYPE`, `constants.ts:113-119` |
| **Event type** | Numeric discriminator: 1=pageView, 2=customEvent, 3=linkEvent, 4=pixelEvent, 5=performance. | `constants.ts:113-119`; derived at `send/route.ts:217-223` |
| **Pageview** | Event type 1 — the default event when no custom `name` is present. | `send/route.ts:221-223` |
| **Custom event** | Event type 2 — a named event (`umami.track('signup')`), optionally with `data`. | `tracker/index.js:207`; `EVENT_TYPE.customEvent` |
| **Session** | A *visitor* over the salt window (default month). Identity = hash(source, ip, ua, salt). One `Session` row per derived id. | `Session` model, `schema.prisma:34`; derivation `send/route.ts:147` |
| **Visit** | A *sitting* within a session, expiring after 30 min of inactivity. Many visits per session. | `visitId` derivation, `send/route.ts:168-175` |
| **Visitor / unique visitor** | Distinct `session_id` count. "Unique visitors" = `count(distinct session_id)`. | `getWebsiteStats.ts:44` (`visitors`) |
| **Bounce** | A visit with exactly one pageview. | `getWebsiteStats.ts:38`; `getExcludeBounceQuery` `prisma.ts:180-191` |
| **distinctId** | An *explicit* user id supplied via `umami.identify(id)`, overriding the hashed identity. | `send/route.ts:147` (`id`), `SessionData.distinctId` `schema.prisma:183` |

> **Confusing pair — session vs. visit.** A *session* is the person (over the salt window); a *visit* is one continuous sitting (30-min timeout). One session can contain many visits. Interviewers love "how do you count returning visitors vs. sessions?" — answer with these two anchors.

## Entities you can track & share

| Term | Meaning | Code anchor |
| --- | --- | --- |
| **Website** | The primary tracked entity; owns events, sessions, reports. | `Website` model, `schema.prisma:66` |
| **Entity / entity type** | The generic notion covering website(1)/link(2)/pixel(3)/board(4) for sharing & permissions. | `ENTITY_TYPE`, `constants.ts:121-126`; `getEntity`, `lib/entity.ts` |
| **Link** | A trackable short link (`/q/[slug]`) that records a click then redirects. | `Link` model `schema.prisma:288`; `(collect)/q/[slug]/route.ts` |
| **Pixel** | A tracking pixel (`/p/[slug]`) that records an open then returns a 1×1 GIF (email tracking). | `Pixel` model `schema.prisma:309`; `(collect)/p/[slug]/route.ts` |
| **Board** | A dashboard composed of components across websites/pixels/links. | `Board` model `schema.prisma:329`; `lib/boards.ts` |
| **Share** | A public, slug-addressed grant to view an entity without an account. | `Share` model `schema.prisma:348`; `api/share/[slug]/route.ts` |

> **Confusing pair — website vs. entity.** In permission and share code, `websiteId` is often really an *entity* id (a pixel or link id can flow through `canViewWebsite`, see `permissions/website.ts:12-21`). Don't assume `websiteId` means a `Website`.

## Reporting & segmentation

| Term | Meaning | Code anchor |
| --- | --- | --- |
| **Report** | A saved analysis of a type: funnel, retention, attribution, goal, journey, revenue, UTM, breakdown, performance. | `Report` model `schema.prisma:232`; `reportTypeSchema` `schema.ts:279-289` |
| **Funnel** | Ordered steps; measures drop-off across them within a time window. | `getFunnel.ts` |
| **Retention** | Cohort return rate over time. | `getRetention.ts` |
| **Attribution** | Credit assignment (first-/last-click) to touchpoints. | `attributionReportSchema` `schema.ts:250-260` |
| **Segment** | A saved filter set applied to queries. | `Segment` model `schema.prisma:253`; merged in `request.ts:123-132` |
| **Cohort** | A segment defined by an action, joined into queries as a subquery. | `getQueryFilters` cohort branch `request.ts:134-160`; `getCohortQuery` `prisma.ts:152-173` |
| **Filter / match** | Column filters (browser, path, country...) combined with `all`/`any`. | `filterParams` `schema.ts:45-71`; `getFilterQuery` `prisma.ts:108-150` |

## Identity, auth & tenancy

| Term | Meaning | Code anchor |
| --- | --- | --- |
| **User / role** | An account with a role: admin, user, view-only. | `User` model `schema.prisma:12`; `ROLES` `constants.ts:164-172` |
| **Team / team role** | A group; members have team roles (owner, manager, member, view-only). | `Team`/`TeamUser` `schema.prisma:197-230`; roles `constants.ts:168-171` |
| **Permission** | A capability string (`website:create`, `team:delete`, ...) mapped to roles. | `PERMISSIONS`/`ROLE_PERMISSIONS` `constants.ts:174-217` |
| **Share token** | A signed, scoped token granting entity access without login. | `parseShareToken` `auth.ts:80`; scope check `permissions/website.ts:12-21` |
| **Cache token** | The `x-umami-cache` JWT that lets `/api/send` skip work. Not an auth token. | `send/route.ts:104-112,311` |
| **Auth key** | A Redis key backing a stateful session (when Redis is enabled). | `saveAuth` `auth.ts:62-74` |

## Collection-time terms

| Term | Meaning | Code anchor |
| --- | --- | --- |
| **Collection type** | The `type` on `/api/send`: `event`, `identify`, `performance` (and `record`). | `COLLECTION_TYPE`, `constants.ts:99-104` |
| **Salt / salt rotation** | The time-bucketed value hashed into identity; rotates day/week/month for privacy. | `getSalt` `crypto.ts:72-78`; `SALT_ROTATION` env |
| **Web Vitals (lcp/inp/cls/fcp/ttfb)** | Performance metrics captured by the tracker's `PerformanceObserver`s. | `tracker/index.js:231-383`; thresholds `constants.ts:106-112` |
| **UTM / click id** | Campaign params (`utm_*`) and ad click ids (`gclid`, `fbclid`, ...) parsed from the URL. | `send/route.ts:191-203` |
| **Data type (EAV)** | Numeric tag (1=string...5=array) telling how a custom-event property is stored. | `DATA_TYPE` `constants.ts:128-134`; used in `saveEventData.ts` |

## Near-synonyms to keep straight

- **"visitors" vs "visits" vs "pageviews"**: distinct sessions vs. distinct visits vs. total type-1 events. All computed in one query — `getWebsiteStats.ts:43-49`.
- **"identify" vs "track"**: `identify` attaches session data / a `distinctId`; `track` records an event. Both go to `/api/send` with different `type`. `tracker/index.js:207-227`.
- **"authenticate" vs "authorize"**: `lib/auth.ts` answers *who are you*; `permissions/` answers *may you*. Keep them separate in your speech — interviewers listen for it.

**Drill:** For each of these product sentences, name the model(s) and the code path: (a) "a returning visitor triggered a purchase event worth $40," (b) "someone opened the newsletter," (c) "a teammate with view-only access looked at the funnel." Check yourself against the tables above.
