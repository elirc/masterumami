# Observability & Operations

"How would I know this broke?" is a senior question, and Umami's honest answer exposes both what's present and a real gap. This page maps logging, health, deploy/rollback, and per-flow failure visibility.

## What's present

| Capability | Mechanism | Anchor |
| --- | --- | --- |
| **Namespaced debug logs** | `debug` library, `umami:*` namespaces (auth, prisma, redis, clickhouse, kafka) | `debug('umami:auth')` [`auth.ts:9`](../../../src/lib/auth.ts) |
| **SQL logging** | `LOG_QUERY=1` logs query + params + name | [`prisma.ts:256-260`](../../../src/lib/prisma.ts) |
| **Error serialization** | `serializeError` for loggable errors | [`send/route.ts:314`](../../../src/app/api/send/route.ts) |
| **Health check** | `GET /api/heartbeat`, used by container healthcheck | [`docker-compose.yml`](../../../docker-compose.yml) |
| **DB preflight** | `check-db.js` verifies connection + min version at build | [`scripts/check-db.js`](../../../scripts/check-db.js) |
| **Telemetry (opt)** | `scripts/telemetry.js`, `api/scripts/telemetry` | — |

Turn on logs locally: `DEBUG=umami:* pnpm dev`, or `LOG_QUERY=1` to see every SQL statement. These are your primary in-repo debugging tools ([Systematic Debugging](03-systematic-debugging.md)).

## What's missing (the honest gap)

There are **no metrics, no tracing, and no alerting** in the repo. Failures surface only as log lines:

- A dropped event on the Kafka path logs a single `KAFKA ERROR` line ([`kafka.ts:89`](../../../src/lib/kafka.ts)) and returns 200 — invisible unless someone greps logs.
- Collection 500s `console.log` a serialized error ([`send/route.ts:318`](../../../src/app/api/send/route.ts)) — no counter, no rate, no page.
- The heartbeat is shallow (it doesn't verify the DB is writable, just that the process answers).

So the truthful answer to "how would you know events are being dropped?" today is: *you wouldn't, until a metrics gap or a customer complaint surfaced it.* That's the observability project in [`../06-contribution-practice/03-senior-build-projects.md`](../06-contribution-practice/03-senior-build-projects.md).

## "How would I know it broke?" per flow

| Flow | Failure | Detectable today? | What you'd add |
| --- | --- | --- | --- |
| Collection (Postgres) | insert fails | Yes — 500 logged, client sees error | error-rate metric |
| Collection (Kafka) | broker down | **No** — 200 returned, one log line | accepted-vs-persisted reconciliation metric + alert |
| Read/metrics | slow query | Only via `LOG_QUERY`/APM | query-duration histogram |
| Auth | token invalid | Client 401 (expected) | auth-failure rate (spike = attack) |
| Replay ingest | save fails | 500 logged | ingest error metric |
| Cache | Redis down | Degrades (queries fall through) | cache hit-rate + connection alert |

The most important row is Kafka collection: it's the one failure that's *silent and data-losing*, which is why it's both a reliability and an observability priority.

## Deploy & rollback

- **Artifacts:** Docker image (`ghcr.io/umami-software/umami`, [`docker-compose.yml`](../../../docker-compose.yml)); also Netlify (`netlify.toml`) and standard Node (`server.js`, `start-docker`).
- **DB migrations:** `prisma migrate deploy` via `update-db`, run before serving (`start-docker` chains `check-db` → `update-tracker` → `start-server`, [`package.json:18`](../../../package.json)). Migrations are **forward-only** (`prisma/migrations/`).
- **Rollback consideration:** because migrations are forward-only, a schema change is not trivially reversible. **Blast radius** thinking: additive migrations (nullable columns) are safe to roll back (old code ignores the column); destructive ones (drop/rename/narrow) are not — rolling back the app leaves it reading a changed schema. Prefer additive, multi-step migrations (expand → migrate → contract) for anything risky.
- **Tracker update on deploy:** `update-tracker` runs at container start ([`package.json:32`](../../../package.json)) so the served `script.js` matches the deployed version.

## Operational levers (env-driven)

You can change runtime behavior without a code deploy via env (see [`../01-codebase-cartography/04-runtime-and-tooling-map.md`](../01-codebase-cartography/04-runtime-and-tooling-map.md)):
- `IGNORE_IP` to block abusive IPs immediately.
- `DISABLE_BOT_CHECK`, `SKIP_LOCATION_HEADERS` to toggle filtering.
- `LOG_QUERY`, `DEBUG` to increase visibility during an incident.
- `DATABASE_REPLICA_URL` to offload reads.

Knowing which knobs exist is operational literacy — in an incident, reaching for `IGNORE_IP` or `LOG_QUERY` beats a hotfix deploy.

## Drill

For each flow in the table, write the one metric you'd add and the alert threshold. Then answer: if events silently stopped persisting on a Kafka deployment, what's the *first* signal you'd want, and why isn't it there today? (Answer: an accepted-vs-persisted reconciliation gap; today only a log line exists.)

## Interview angle

- "How would you know if this broke in production?" → per-flow table; name the silent Kafka gap.
- "What would you add to make this observable?" → metrics for accepted/persisted/error-rate, reconciliation, structured logs with ids.
- "How do you deploy DB changes safely?" → forward-only migrations, expand/contract, additive-first, rollback blast radius.
- "What can you change without a deploy?" → the env levers.
