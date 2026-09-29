# Runtime and Tooling Map

Everything you need to build, run, test, and reason about *where code executes*. Commands are `__verified__` (run here) or `__inferred__` (read from config; this clone has no deps/DB installed).

## Toolchain at a glance

| Concern | Tool | Evidence |
| --- | --- | --- |
| Package manager | **pnpm** | `pnpm-lock.yaml`, `pnpm-workspace.yaml` __verified__ |
| Framework | **Next.js 16** (App Router, Turbopack) | `next: 16.2.6`, scripts use `--turbo` (`package.json:14,21`) __verified__ |
| Language | **TypeScript 5.9**, `"type": "module"` (ESM) | `package.json:12,158` __verified__ |
| UI | **React 19** + React Compiler (`babel-plugin-react-compiler`) | `package.json:101,136` __verified__ |
| ORM / DB | **Prisma 7** + `@prisma/adapter-pg`; Postgres | `package.json:60-61`, `prisma/schema.prisma:7-10` __verified__ |
| Analytics DB | **ClickHouse** (`@clickhouse/client`) | `package.json:55` __verified__ |
| Cache | **Redis** (optional) | `package.json:108`, `src/lib/redis.ts` __verified__ |
| Queue | **Kafka** (`kafkajs`, optional) | `package.json:90`, `src/lib/kafka.ts` __verified__ |
| Validation | **Zod 4** | `package.json:117` __verified__ |
| Client state | **Zustand** (global) + **React Query** (server state) | `package.json:65,119` __verified__ |
| Lint/format | **Biome** | `biome.json`, scripts `lint/format/check` (`package.json:47-49`) __verified__ |
| Unit tests | **Vitest 4** + Testing Library + jsdom | `package.json:159,130-131,140` __verified__ |
| e2e tests | **Cypress** | `cypress/`, `cypress.config.ts` __verified__ |
| i18n | **next-intl** | `package.json:94`, `src/i18n/` __verified__ |
| Bundlers (scripts) | **Rollup** (tracker/recorder), **tsup** (components) | `rollup.*.config.js`, `tsup.config.js` __verified__ |

## The build pipeline

`pnpm build` runs, in order (`package.json:15`): `check-env` → `build-db` → `check-db` → `build-tracker` → `build-recorder` → `build-geo` → `build-app`. Reading that list tells you the app depends on: a validated env, a generated Prisma client, a compiled tracker + recorder, and a downloaded GeoLite DB — before Next even builds.

| Command | Does | Status |
| --- | --- | --- |
| `pnpm install` | Install deps | __inferred__ |
| `pnpm dev` | `dotenv next dev --turbo` (`package.json:14`) | __inferred__ |
| `pnpm build` | Full production build (chain above) | __inferred__ |
| `pnpm build-db` | `prisma generate` + build prisma client (`package.json:29`) | __inferred__ |
| `pnpm update-db` | `prisma migrate deploy` (`package.json:33`) | __inferred__ |
| `pnpm test` | `vitest run` (`package.json:42`) | __inferred__ |
| `pnpm test:watch` | `vitest` | __inferred__ |
| `pnpm cypress-open` | Open Cypress runner | __inferred__ |
| `pnpm lint` / `format` / `check` | Biome | __inferred__ |
| `pnpm seed-data` | `tsx scripts/seed-data.ts` | __inferred__ |
| `pnpm build-tracker` | Rollup-build `src/tracker` → `public/script.js` | __inferred__ |

## Where code executes (runtime boundaries)

This is the map that prevents "why is `process.env` undefined?" confusion.

| Boundary | Runs on | Examples | Constraints |
| --- | --- | --- | --- |
| **Visitor browser (tracker)** | Any site embedding `script.js` | `src/tracker/index.js`, `src/recorder/index.js` | No Node APIs; must be tiny; `keepalive` fetch; no secrets |
| **Server (route handlers)** | Node (Next server) | `src/app/api/**`, `src/app/(collect)/**`, everything in `src/lib` touching DB | Full Node; secrets OK; DB clients live here |
| **Dashboard browser (React)** | User's browser | `src/app/(main)/**`, `src/components/**` | No DB, no `APP_SECRET`; only `process.env` values exposed via `next.config.ts` |
| **Build/scripts** | Node CLI | `scripts/*.js`, prisma generate | One-shot; not in the request path |

A telling detail: config values needed in the *browser* are injected through `next.config.ts` (e.g. `basePath`, `defaultLocale`) rather than read from `process.env` at runtime — client code can't see server env. And the tracker is templated at build time (`__COLLECT_API_HOST__`, `tracker/index.js:43`), not configured at runtime.

## Env vars (high level — no secrets)

Read at `next.config.ts` (top block) and across `src/lib/`. Grouped by concern:

| Group | Vars | Effect |
| --- | --- | --- |
| **Required** | `DATABASE_URL`, `APP_SECRET` | DB connection; token signing/encryption secret (`crypto.ts:56-57`) |
| **Storage backends** | `CLICKHOUSE_URL`, `KAFKA_URL`+`KAFKA_BROKER`, `REDIS_URL`, `DATABASE_REPLICA_URL` | Switch persistence/caching paths (`db.ts`, `kafka.ts`, `redis.ts`, `prisma.ts:372`) |
| **Collection behavior** | `SALT_ROTATION`, `IGNORE_IP`, `DISABLE_BOT_CHECK`, `REMOVE_TRAILING_SLASH`, `SKIP_LOCATION_HEADERS` | Tune identity/filtering (`send/route.ts`, `detect.ts`) |
| **Deployment/cloud** | `CLOUD_MODE`, `CLOUD_URL`, `BASE_PATH`, `FORCE_SSL`, `ALLOWED_FRAME_URLS` | Cloud gating, CSP, framing (`next.config.ts`, `detect.ts`, `request.ts`) |
| **Tracker** | `TRACKER_SCRIPT_NAME`, `TRACKER_SCRIPT_URL`, `COLLECT_API_ENDPOINT` | Rename/relocate the script and endpoint |
| **Debug** | `LOG_QUERY`, `DEBUG=umami:*` | Log SQL / namespace logs (`prisma.ts:256`) |

Never commit real values; `check-env.js` validates required ones at build (`scripts/check-env.js`).

## Local run, the fast way

`docker compose up` starts Postgres 15 + the app with a `/api/heartbeat` healthcheck (`docker-compose.yml`). For a from-source run you need at minimum `DATABASE_URL` + `APP_SECRET`, then `pnpm build-db && pnpm dev`. ClickHouse/Redis/Kafka are all optional — the app runs on Postgres alone. __inferred__

## Senior noticing

- The persistence path is chosen entirely by **environment variables**, not code — so "which database is this using?" is answered by the deployment, not the repo. That's why every query file has two branches.
- The tracker being **build-time templated** means changing the collect endpoint is a rebuild, not a config toggle — a deployment constraint worth knowing.
- The build chain front-loads `check-env`/`check-db`: the app refuses to build against a misconfigured or out-of-date database, a nice fail-fast.

**Drill:** You're handed a running Umami and asked "is it using ClickHouse or Postgres, and is Redis on?" Name the exact env vars you'd check and the code that reads each (`db.ts:23`, `redis.ts:106`). Then say what changes in the request path for each.
