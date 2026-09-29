# Security Checklist

Every major web-security concern mapped to Umami's real code, with a pre-merge checklist at the end. Confirmed protections and real gaps are both labelled. This is also the security-interview crib sheet.

## Authorization / IDOR — the #1 concern

**Protection:** authenticate in `parseRequest`, then an explicit `canViewWebsite`/`canUpdate*`/`canDelete*` per handler ([`permissions/website.ts:7`](../../src/permissions/website.ts), called e.g. [`websites/[websiteId]/route.ts:28`](../../src/app/api/websites/[websiteId]/route.ts)).

**Gap:** the check is manual and per-route. A new `[id]` route that forgets it is an IDOR. There's no systemic enforcement. **Mitigation:** the `withAuthorization` wrapper / lint-rule idea in the [architecture critique](../03-architecture-and-patterns/06-architecture-critique.md).

**Check:** every id-bearing route calls a `can*` predicate against the URL id before querying.

## Input validation

**Protection:** Zod schemas at every route via `parseRequest` ([`request.ts:23-42`](../../src/lib/request.ts)); `schema.ts` holds the contracts; cross-field `.refine`s enforce invariants; write-time `substring` truncation guards column widths ([`saveEvent.ts:113-131`](../../src/queries/sql/events/saveEvent.ts)).

**Gap:** suffixed filters (`browser1`) are re-added *after* Zod, unvalidated ([`request.ts:33-38`](../../src/lib/request.ts)); report `parameters` use `anyObjectParam` in the save path.

**Check:** all external input passes a schema; no field bypasses validation before reaching a query or the DB.

## SQL injection

**Protection:** the raw-SQL engine binds values via `{{param}}`→`$n` ([`prisma.ts:268-276`](../../src/lib/prisma.ts)); ClickHouse uses `{param:Type}` binding. Values are never concatenated into SQL.

**Gap (possible risk, labelled):** `pagedRawQuery` interpolates `order by ${orderBy}` directly ([`prisma.ts:318-323`](../../src/lib/prisma.ts)). Not currently reachable with user input, but a latent injection surface. **Mitigation:** allow-list sortable columns (Ticket 6, critique #4).

**Check:** no user value inside a SQL template string; only bound parameters. Any dynamic identifier (column/table/order) comes from an allow-list.

## XSS (cross-site scripting)

**Protection:** React escapes rendered values by default. A **Content-Security-Policy** is set in `next.config.ts` (`default-src 'self'`, restricted `script-src`/`frame-ancestors`).

**Considerations:** analytics displays user/site-controlled strings (page titles, URLs, referrers, custom event data). As long as these render as React text (not `dangerouslySetInnerHTML`), they're escaped. **Check:** no `dangerouslySetInnerHTML` with event/site-derived data; the CSP stays restrictive. Note the CSP includes `'unsafe-inline'`/`'unsafe-eval'` for scripts (`next.config.ts`) — a loosening worth knowing when discussing hardening.

## CSRF

**Protection:** the API is authenticated by a **bearer token in the `Authorization` header** (from localStorage, [`useApi.ts:31`](../../src/components/hooks/useApi.ts)), not a cookie. CSRF fundamentally relies on the browser auto-attaching credentials (cookies); a header token isn't auto-attached, so classic CSRF doesn't apply to the authed API.

**Tradeoff:** the cost of that CSRF-immunity is XSS exposure — a token in localStorage is readable by any injected script. So the security model trades CSRF risk for XSS risk; the CSP is what backstops the XSS side. Being able to state that tradeoff is a strong interview answer.

**Check:** authed endpoints rely on the header token, not ambient cookies.

## SSRF (server-side request forgery)

**Consideration:** the server makes outbound requests for geo/country-name downloads (build scripts) and, in cloud mode, reads proxy headers. The collection endpoint parses user-supplied URLs (`new URL(url, base)`, [`send/route.ts:179`](../../src/app/api/send/route.ts)) but only to *parse* them, not to fetch them — so no SSRF from event URLs. **Check:** if you ever add a feature that *fetches* a user-supplied URL (webhooks, link previews), add SSRF protections (block internal ranges).

## Secrets management

**Protection:** `APP_SECRET` drives token signing/encryption ([`crypto.ts:56-57`](../../src/lib/crypto.ts)); `check-env.js` validates required env at build; secrets live in env, not code. Tokens carrying user identity are *encrypted*, not just signed ([`jwt.ts:16`](../../src/lib/jwt.ts)).

**Check:** no secret in code/logs; `serializeError` output isn't returned to clients in a way that leaks config (it currently can — see Error handling).

## Passwords & auth

**Protection:** bcrypt hashing (`checkPassword`/`hashPassword`, `lib/password.ts`); uniform bad-user/bad-password error to prevent enumeration ([`login/route.ts:28-29`](../../src/app/api/auth/login/route.ts)); PBKDF2 for token key derivation ([`crypto.ts:16`](../../src/lib/crypto.ts)).

**Gap:** no rate limiting on login (`redis.rateLimit` exists, uncalled) — brute-force exposure. **Check:** consider IP-based login throttling (Review Kata 5).

## Bot & abuse filtering

**Protection:** `isbot(userAgent)` drops bot traffic ([`send/route.ts:131`](../../src/app/api/send/route.ts)); `hasBlockedIp` supports exact + CIDR IP blocking ([`detect.ts:140-170`](../../src/lib/detect.ts)).

**Gap:** no rate limiting on the open collection endpoints; `/batch` amplifies (Ticket 10). **Check:** ingest has volume protection (app or infra).

## Information disclosure

**Gap (labelled):** collection catch blocks return `serializeError(e)` to the client ([`send/route.ts:320`](../../src/app/api/send/route.ts)) — a serialized error (potentially stack/query fragments) sent to an unauthenticated caller. **Mitigation:** log server-side, return a generic message + id (Review Kata 4).

## Data trust (analytics-specific)

**Consideration:** the entire `/api/send` payload is attacker-controlled, including `revenue`, `ip`/`userAgent` overrides, and geo headers. Values like revenue are only as trustworthy as the embedding site. **Check:** never treat client-supplied analytics values as authoritative for anything security- or money-critical.

## Cookies / sessions

**Protection:** stateless encrypted tokens or Redis-backed sessions ([Flow 3](../01-codebase-cartography/05-key-flows.md#flow-3-login-and-the-two-token-strategies)). **Consideration:** stateless tokens can't be revoked before expiry — logout only clears localStorage. **Check:** if revocation matters, use the Redis path.

## Dependencies & uploads

- **Uploads:** session replay stores rrweb event bytes ([`schema.prisma:368`](../../prisma/schema.prisma)); capped at 200 events/request. No arbitrary file upload surface found.
- **Dependencies:** pnpm lockfile pins versions; standard `pnpm audit` applies.

---

## Pre-merge security checklist

Before merging any change that touches a route, query, or auth:

- [ ] Every id-bearing route calls a `can*` authorization check against the URL id.
- [ ] All new external input passes a Zod schema; nothing bypasses it.
- [ ] No user value concatenated into SQL; dynamic identifiers come from an allow-list.
- [ ] No `dangerouslySetInnerHTML` with user/site-derived data.
- [ ] No secret in code, logs, or client-returned errors.
- [ ] Error responses to clients are generic; full detail logged server-side.
- [ ] New outbound-fetch features validate the target (SSRF).
- [ ] New public/unauth endpoints have volume protection.
- [ ] Auth changes preserve uniform error messages (no enumeration).
- [ ] Tests cover the deny path (cross-tenant 401, invalid input 400).

## Interview angle

- "Walk me through the OWASP-ish risks in an app like this." → use the sections above with anchors.
- "Cookie vs. header auth — security tradeoffs?" → CSRF-immunity for XSS-exposure, backstopped by CSP.
- "How do you prevent IDOR at scale?" → per-handler `can*` + a systemic guardrail.
- "Is raw SQL safe?" → bound params yes; dynamic identifiers need allow-lists — cite the `orderBy` surface.
