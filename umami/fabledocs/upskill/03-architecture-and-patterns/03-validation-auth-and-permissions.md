# Validation, Authentication & Authorization

Three distinct concerns that juniors blur together. This page maps all three in Umami, with the multi-tenant isolation model and the IDOR surface.

- **Validation:** is the input well-formed? (Zod)
- **Authentication (authN):** who are you? (`lib/auth`)
- **Authorization (authZ):** may *you* do *this* to *this resource*? (`permissions/`)

## The pipeline, once more

Every authed route runs the same shape ([`request.ts:12-53`](../../src/lib/request.ts)):

```
parseRequest → Zod validate → checkAuth (authN)   [in lib/request]
handler      → can*(auth, resourceId) (authZ)      [in the route]
             → query
```

Validation and authentication are centralized; **authorization is per-handler** and that's the crux of the security model.

## Layer 1 — Validation (all the places)

| Where | What it validates | Anchor |
| --- | --- | --- |
| Route Zod schema | Body/query shape, types, bounds, cross-field invariants | `schema.ts`, per-route inline schemas |
| `parseRequest` | Runs the schema, coerces, handles GET vs POST | `request.ts:23-42` |
| `.refine`/`.superRefine` | Invariants (exactly-one-source, date-range presence) | `send/route.ts:53`, `schema.ts:32` |
| Write-time truncation | Column-width safety (`substring`) | `saveEvent.ts:113-131` |
| DB constraints | Uniqueness (`slug`, `username`), FK (Prisma-enforced) | `schema.prisma` |

The lesson: validation happens in *layers*, and the strongest is the one closest to entry (Zod). Write-time truncation is a second belt-and-suspenders layer because the public collection payload can't be fully trusted.

Known gap: the suffixed-filter re-add ([`request.ts:33-38`](../../src/lib/request.ts)) lets some values skip Zod. And the report save path uses `anyObjectParam` for `parameters` while the strict discriminated union is applied elsewhere ([`schema.ts:276` vs `:279`](../../src/lib/schema.ts)) — verify strict validation actually runs on writes.

## Layer 2 — Authentication

`checkAuth` ([`auth.ts:17-60`](../../src/lib/auth.ts)) resolves identity from one of three credentials:

1. **Bearer token** — `parseSecureToken` decrypts+verifies an encrypted JWT carrying `userId` (stateless) or `authKey` (Redis-backed session).
2. **Redis auth key** — when the token carries an `authKey` and Redis is enabled, the session is looked up server-side (revocable).
3. **Share token** — a signed, scoped token for public dashboard access, gated by a context header ([`auth.ts:42-48`](../../src/lib/auth.ts)).

Two token strategies from one login ([`login/route.ts:36-40`](../../src/app/api/auth/login/route.ts)); see [Flow 3](../01-codebase-cartography/05-key-flows.md#flow-3-login-and-the-two-token-strategies). The admin flag is derived here (`user.isAdmin = user.role === ROLES.admin`, [`auth.ts:51`](../../src/lib/auth.ts)).

Passwords: bcrypt via `checkPassword` ([`login/route.ts:28`](../../src/app/api/auth/login/route.ts), `lib/password.ts`), with a uniform error for bad-user vs. bad-password to avoid username enumeration.

## Layer 3 — Authorization

Two mechanisms compose:

### Role → permission matrix (capability check)

`ROLE_PERMISSIONS` maps each role to allowed permission strings ([`constants.ts:186-217`](../../src/lib/constants.ts)); `hasPermission(role, perm)` checks membership ([`auth.ts:76-78`](../../src/lib/auth.ts)). E.g. a `user` may `website:create` but not `team:delete`.

### Resource-ownership predicates (instance check)

`permissions/website.ts` etc. answer "may this auth touch *this* resource," combining admin bypass, share-token scope, direct ownership, and team-role checks ([`website.ts:7-40`](../../src/permissions/website.ts)). `canUpdateWebsite` combines *both*: team membership **and** `hasPermission(teamUser.role, PERMISSIONS.websiteUpdate)` ([`website.ts:77-81`](../../src/permissions/website.ts)).

So authorization = "do you have the capability (role)?" AND "do you own/belong-to the instance?" Being able to separate capability from instance authorization is a senior distinction.

## Multi-tenancy & isolation

Isolation is enforced *in every query* by scoping to `website_id`, plus the `can*` gate before the query. Three isolation mechanisms:

1. **Ownership**: a website has `userId` XOR `teamId`; access requires matching one ([`website.ts:29-37`](../../src/permissions/website.ts)).
2. **Team membership**: `getTeamUser(teamId, userId)` must exist ([`website.ts:34-36`](../../src/permissions/website.ts)).
3. **Optional DB schema isolation**: a `?schema=` in the connection URL sets Postgres `search_path` per request ([`prisma.ts:262-266,364-368`](../../src/lib/prisma.ts)) — enabling schema-per-tenant deployments.

## IDOR: the primary risk class

An **IDOR** (Insecure Direct Object Reference) is when a user accesses another's resource by supplying its id, because the server checked *authentication* but not *authorization for that id*. Umami's defense is the `canViewWebsite(auth, id)` call in each handler. Its weakness is that the call is **manual and per-route** — a new route that forgets it is an IDOR.

To recognize/prevent:
- Every `[id]` route must call a matching `can*` predicate *before* querying. Audit by grepping the route for `canView`/`canUpdate`/`canDelete`.
- The predicate must check against the id from the URL, not a client-supplied field.
- Share tokens must be scoped (they are — [`website.ts:12-21`](../../src/permissions/website.ts)) and context-gated (they are — [`auth.ts:43-47`](../../src/lib/auth.ts)).

"What a junior misses vs. what a senior checks": a junior verifies the happy path (owner can view). A senior tries `GET /websites/<someone-else's-id>` with a valid token and confirms a 401, then asks whether *every* route does the check — because the systemic risk isn't this route, it's the next one.

## Drills

1. **Capability vs instance:** In `canUpdateWebsite` (`website.ts:58-84`), identify the line doing the *instance* check and the line doing the *capability* check. Why are both needed?
2. **IDOR audit:** Pick five `[id]` routes at random. For each, confirm the authz call exists and targets the URL id. Report any gap as a risk-register entry.
3. **Token scope:** A leaked share token is replayed against a non-share API route. Trace `auth.ts:42-48` and state exactly what stops it.

## Interview angle

- "AuthN vs authZ — draw the line." → `lib/auth` vs `permissions/`.
- "What's an IDOR and how do you prevent it systematically?" → per-handler `can*`, plus the guardrail idea.
- "Role-based vs ownership-based authorization?" → Umami uses both, composed.
- "How do you isolate tenants?" → per-query `website_id` scoping + optional schema-per-tenant.
See [`../08-interview-prep/03-api-and-data-modeling-questions.md`](../08-interview-prep/03-api-and-data-modeling-questions.md) and [`../05-quality-engineering/05-security-checklist.md`](../05-quality-engineering/05-security-checklist.md).
