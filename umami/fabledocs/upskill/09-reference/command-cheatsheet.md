# Command Cheatsheet

Every command marked `__verified__` (run in this environment) or `__inferred__` (read from `package.json`/config but not executed — this clone has no `node_modules`, `.env`, or database). All script sources are `package.json` unless noted.

## Setup

| Command | Purpose | Status |
| --- | --- | --- |
| `pnpm install` | Install dependencies | __inferred__ |
| `cp .env.example .env` | Create env file (set `DATABASE_URL`, `APP_SECRET`) | __inferred__ |
| `pnpm build-db` | `prisma generate` + build prisma client (`:29`) | __inferred__ |
| `docker compose up` | Postgres 15 + app with heartbeat healthcheck (`docker-compose.yml`) | __inferred__ |

## Develop

| Command | Purpose | Status |
| --- | --- | --- |
| `pnpm dev` | Dev server (`dotenv next dev --turbo`, `:14`) | __inferred__ |
| `DEBUG=umami:* pnpm dev` | Dev with namespaced debug logs | __inferred__ |
| `LOG_QUERY=1 pnpm dev` | Log every SQL query + params (`prisma.ts:256`) | __inferred__ |
| `pnpm build` | Full prod build chain (`:15`) | __inferred__ |
| `pnpm start` | Serve the production build (`next start`) | __inferred__ |

## Build sub-steps (the chain)

| Command | Purpose | Status |
| --- | --- | --- |
| `pnpm build-app` | `next build --turbo` (`:21`) | __inferred__ |
| `pnpm build-tracker` | Rollup-build `src/tracker` → `public/script.js` (`:24`) | __inferred__ |
| `pnpm build-recorder` | Rollup-build the recorder (`:25`) | __inferred__ |
| `pnpm build-geo` | Download GeoLite DB (`:28`) | __inferred__ |
| `pnpm build-components` | tsup the shareable component package (`:23`) | __inferred__ |
| `pnpm check-env` | Validate required env (`scripts/check-env.js`, `:35`) | __inferred__ |
| `pnpm check-db` | Verify DB connection + min version (`scripts/check-db.js`, `:34`) | __inferred__ |

## Database

| Command | Purpose | Status |
| --- | --- | --- |
| `pnpm build-db-client` | `prisma generate` (`:31`) | __inferred__ |
| `pnpm build-db-schema` | `prisma db pull` (`:30`) | __inferred__ |
| `pnpm update-db` | `prisma migrate deploy` (`:33`) | __inferred__ |
| `pnpm seed-data` | Seed sample data (`tsx scripts/seed-data.ts`, `:46`) | __inferred__ |
| `pnpm change-password` | Reset a user password (`scripts/change-password.js`, `:40`) | __inferred__ |

## Test

| Command | Purpose | Status |
| --- | --- | --- |
| `pnpm test` | Run unit/component tests (`vitest run`, `:42`) | __inferred__ |
| `pnpm test <path>` | Run one test file | __inferred__ |
| `pnpm test:watch` | Vitest watch mode (`:43`) | __inferred__ |
| `pnpm cypress-open` | Open the Cypress runner (`:44`) | __inferred__ |
| `pnpm cypress-run` | Run Cypress headless (`:45`) | __inferred__ |

## Lint / format

| Command | Purpose | Status |
| --- | --- | --- |
| `pnpm lint` | Biome lint (`biome lint .`, `:47`) | __inferred__ |
| `pnpm format` | Biome format write (`:48`) | __inferred__ |
| `pnpm check` | Biome lint + format write (`:49`) | __inferred__ |
| `pnpm check-missing-messages` | Find untranslated i18n messages (`:36`) | __inferred__ |

## Docker / deploy

| Command | Purpose | Status |
| --- | --- | --- |
| `pnpm build-docker` | Build chain for Docker (no env/db checks, `:17`) | __inferred__ |
| `pnpm start-docker` | `check-db` → `update-tracker` → `start-server` (`:18`) | __inferred__ |
| `pnpm start-server` | `node server.js` (`:20`) | __inferred__ |

## Useful env toggles (no deploy needed)

| Var | Effect | Anchor |
| --- | --- | --- |
| `LOG_QUERY=1` | Log SQL | `prisma.ts:256` |
| `DEBUG=umami:*` | Namespaced logs | `auth.ts:9`, etc. |
| `IGNORE_IP=1.2.3.4,10.0.0.0/8` | Block IPs (exact/CIDR) | `detect.ts:141` |
| `DISABLE_BOT_CHECK=1` | Skip bot filtering | `send/route.ts:131` |
| `SALT_ROTATION=day\|week\|month` | Identity window | `crypto.ts:72` |
| `DATABASE_REPLICA_URL=...` | Route reads to a replica | `prisma.ts:372` |
| `CLICKHOUSE_URL=...` | Use ClickHouse instead of Postgres | `db.ts:23` |
| `REDIS_URL=...` | Enable Redis cache + stateful auth | `redis.ts:106` |

## Verification note

None of the above were executed in this environment (no dependencies/DB installed in the clone). They are transcribed faithfully from `package.json` scripts and config files. Before relying on any, confirm against the current `package.json` — script names occasionally change. See [`verification-log.md`](verification-log.md).
