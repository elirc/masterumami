# Boundaries & Layers

A **boundary** is a line across which responsibility, trust, or ownership changes. Good architecture makes boundaries explicit; bugs and security holes cluster where boundaries leak. This page names Umami's layers, what each owns and must *not* own, and where the boundaries leak in real code.

## The layers

```
HTTP boundary        route.ts handlers        (src/app/api/**, src/app/(collect)/**)
   │  (validate + authenticate)                src/lib/request.ts, src/lib/auth.ts
   ▼
authorization        permissions/ predicates   (src/permissions/**)
   │
   ▼
data access          queries/                  (src/queries/prisma/**, src/queries/sql/**)
   │
   ▼
persistence engines  lib/db, lib/prisma,       (choose backend, run SQL/ORM)
                     lib/clickhouse, lib/kafka, lib/redis
```

Plus two edges: the **visitor scripts** (`src/tracker`, `src/recorder`) and the **dashboard UI** (`src/app/(main)`, `src/components`), which talk to the HTTP boundary from outside.

## What each layer owns (and must not)

| Layer | Owns | Must NOT own | Leak to watch |
| --- | --- | --- | --- |
| Route handler | HTTP shape; orchestration order (validate→auth→authz→query→respond) | Business SQL; authorization *rules* | Handlers that inline SQL or reimplement a permission check |
| `lib/request` + `lib/auth` | Parsing, validation, authentication | Per-resource authorization | — |
| `permissions/` | Authorization rules | HTTP, SQL, DB reads beyond what a rule needs | Predicates that re-fetch entities differently (below) |
| `queries/` | Data access; the two SQL dialects | HTTP status codes, auth | Query files that assume an authorized caller |
| `lib/*` engines | Connections, the SQL template engine | Product concepts | The BigInt patch (a cross-cutting global) |

## Where the boundaries are clean (learn from these)

- **Validate → authenticate is centralized.** Every route funnels through `parseRequest` ([`request.ts:12`](../../../src/lib/request.ts)); no handler re-implements body parsing or token checking. One place to audit.
- **Authorization is its own module.** `permissions/` ([`permissions/website.ts`](../../../src/permissions/website.ts), `team.ts`, etc.) keeps rules out of handlers and out of queries. A handler reads like a checklist: parse, authorize, query, respond ([`websites/[websiteId]/route.ts:20-34`](../../../src/app/api/websites/[websiteId]/route.ts)).
- **Data access is separated by kind.** CRUD entities go through Prisma models (`queries/prisma/`); analytics go through raw SQL (`queries/sql/`). Each query colocates its Postgres and ClickHouse implementations, so the persistence choice never leaks into handlers.

## Where the boundaries leak (real, anchored)

These are genuine, verified seams — labelled as tradeoffs/risks, not asserted bugs.

1. **Authorization must be *remembered* per handler.** Because authz is a manual `can*` call in each route, a new route can silently skip it — the classic IDOR. There's no framework enforcement that every id-bearing route authorizes. Evidence: the pattern is manual everywhere, e.g. [`realtime/[websiteId]/route.ts:20`](../../../src/app/api/realtime/[websiteId]/route.ts). *This is the single most important boundary risk in the repo.*

2. **Two permission predicates re-fetch the entity differently.** `canViewWebsite` reads via `getEntity` ([`website.ts:23`](../../../src/permissions/website.ts)) while `canUpdateWebsite`/`canDeleteWebsite` read via `getWebsite` ([`website.ts:67,95`](../../../src/permissions/website.ts)). Same concept, two data sources — a divergence risk if one is patched and not the other.

3. **Handler-as-function composition couples routes to `send`.** The pixel, link, and batch routes call `send`'s `POST` directly ([Flow 4](../01-codebase-cartography/05-key-flows.md#flow-4-pixel--link-tracking--handler-composition)). The collection *contract* (its Zod schema) now has four callers, three of them internal — a change ripples across the HTTP boundary invisibly.

4. **A cross-cutting global patch.** `BigInt.prototype.toJSON` is monkey-patched in `lib/db.ts` ([`:8-10`](../../../src/lib/db.ts)) — a process-wide side effect living in a data-layer module. It fixes a real problem (BigInt serialization) but its blast radius is every `JSON.stringify` in the app, and its home is arguably wrong.

5. **Validated data is un-stripped with raw values.** `parseRequest` re-adds numeric-suffixed filter keys *after* Zod validation ([`request.ts:33-38`](../../../src/lib/request.ts)), so those values cross the validation boundary unvalidated. The boundary is real but has a hole cut in it.

## The trust boundary (most important to internalize)

The sharpest boundary is **trust**: which data is attacker-controlled?

- **Fully untrusted:** the `/api/send` payload — it comes from a public script on any site ([`send/route.ts:24`](../../../src/app/api/send/route.ts)). Including `revenue`, `ip`/`userAgent` overrides, and geo headers. The Zod schema + bot/IP checks are the only gate.
- **Semi-trusted:** the `x-umami-cache` token — client-held but signed, so its *contents* are server-issued (can't be forged, can be replayed).
- **Trusted:** data already in the DB, and a value that just passed `canViewWebsite`.

Naming what's trusted at each point is the skill. A senior reading `send/route.ts` immediately flags that `payload.revenue` ([`saveEvent.ts:154`](../../../src/queries/sql/events/saveEvent.ts)) is attacker-supplied and unauthenticated — so revenue numbers are only as trustworthy as the embedding site.

## Drills

1. **Boundary audit:** Pick `websites/[websiteId]/reset/route.ts` (or any id route). Mark the exact line where each boundary is crossed: validation, authentication, authorization, data access. Is any missing?
2. **Leak hunt:** Find one handler that does something a *lower* layer should own (e.g. constructs SQL, or encodes an authz rule inline). Propose where it belongs.
3. **Trust labelling:** For each field in the `/api/send` payload, label it trusted/semi/untrusted and name the guard (if any) that protects downstream use.

## Interview angle

- "How do you keep authorization consistent across many endpoints?" → separate `permissions/` module + the risk that it's per-handler.
- "What's a trust boundary and where's yours?" → the unauthenticated collection payload.
- "Show me a boundary leak and how you'd fix it." → any of the five above, with the fix.
- Self-grade: *Basic* names the layers; *Solid* names what each must not own; *Strong* points at a real leak, explains its blast radius, and proposes a fix that follows the existing pattern.
