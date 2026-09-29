# Tooling & Build System

Tooling questions ("how does your build work?", "ESM vs CJS?") are easy points in an interview if you can speak concretely. Umami gives you a real, multi-artifact build to describe.

## ESM everywhere

`package.json` sets `"type": "module"` ([`:12`](../../package.json)) — the whole app is ES modules. Practical consequences you should be able to state:
- `import`/`export`, not `require`/`module.exports`.
- The `@/` path alias → `src/` (see `tsconfig.json`), used everywhere (`import ... from '@/lib/...'`).
- Config files use ESM too (`next.config.ts` uses `import`, top-level).

The transferable point: ESM is the modern default; CJS interop (`require` of ESM, `__dirname`) is where the friction lives. Umami sidesteps most of it by being ESM top to bottom.

## One repo, several build artifacts

`pnpm build` is a **chain**, not a single step ([`package.json:15`](../../package.json)):

```
check-env → build-db → check-db → build-tracker → build-recorder → build-geo → build-app
```

Each produces something different:

| Step | Tool | Output | Why separate |
| --- | --- | --- | --- |
| `build-db` | Prisma | generated client in `src/generated/prisma` | App can't typecheck without it |
| `build-tracker` | **Rollup** (`rollup.tracker.config.js`) | `public/script.js` (one IIFE) | Runs on *other* sites — must be tiny, standalone |
| `build-recorder` | **Rollup** (`rollup.recorder.config.js`) | recorder script (bundles rrweb) | Same — a separate visitor-side artifact |
| `build-geo` | `scripts/build-geo.js` | GeoLite DB | Runtime geo lookup needs the file present |
| `build-app` | **Next.js** (Turbopack) | `.next/` server + client bundles | The dashboard + API |
| `build-components` | **tsup** | shareable component package | Published separately (`package.components.json`) |

The insight to voice in an interview: **different runtime targets need different bundlers.** The dashboard uses Next's bundler; the tracker uses Rollup because it must be a single, dependency-free, minified IIFE dropped into arbitrary pages. Using Next to build the tracker would be the wrong tool.

## Why the tracker is templated, not configured

The tracker embeds placeholders like `__COLLECT_API_HOST__` and `__COLLECT_API_ENDPOINT__` ([`tracker/index.js:43-44`](../../src/tracker/index.js)) that Rollup's replace plugin fills at build time. So the deployed `script.js` has the endpoint baked in — changing it is a rebuild, not an env toggle at runtime. This is a deliberate constraint: the script has no access to server env, so config must be compiled in.

## Turbopack

`dev` and `build-app` pass `--turbo` ([`package.json:14,21`](../../package.json)) — Next's Rust-based bundler, faster than Webpack for dev refresh. Nothing you write changes; it's a build-speed choice. Worth naming as "we use Turbopack for faster HMR."

## Biome instead of ESLint + Prettier

Umami uses **Biome** for both linting and formatting ([`biome.json`](../../biome.json); scripts `lint`/`format`/`check`, [`package.json:47-49`](../../package.json)). Biome is a single fast (Rust) tool replacing the ESLint+Prettier pair. `pnpm check` lints and formats with `--write`. The talking point: consolidating lint+format into one tool cuts config surface and CI time; the tradeoff is a smaller plugin ecosystem than ESLint.

## Testing tools

- **Vitest** for unit tests (`vitest.config.ts`, `.test.ts` files) — fast, ESM-native, Jest-compatible API. Setup in `src/test/setup.ts`, a custom render in `src/test/render.tsx`.
- **Testing Library** + **jsdom** for component tests.
- **Cypress** for e2e against a running app, selecting by `data-test` attributes.

See [`../05-quality-engineering/01-testing-strategy.md`](../05-quality-engineering/01-testing-strategy.md) for how these layers divide work.

## Prisma tooling specifics

- `prisma generate` builds the typed client; `prisma migrate deploy` applies migrations (`update-db`).
- The schema uses `relationMode = "prisma"` ([`schema.prisma:9`](../../prisma/schema.prisma)) — foreign-key enforcement is done by Prisma, not the DB, which matters for how referential integrity behaves (and is a deliberate choice for multi-DB portability).
- `prisma.config.ts` and `tsconfig.prisma.json` configure the generation.

## Pitfall checklist (tooling)

- [ ] Editing `public/script.js` directly — it's generated; edit `src/tracker/index.js` and rebuild.
- [ ] Expecting a tracker config change to take effect without a rebuild.
- [ ] Running the app before `build-db` (no generated client → type/import errors).
- [ ] Assuming ESLint config — it's Biome.
- [ ] Assuming DB-level FK constraints — `relationMode="prisma"` moves them to the app.

## Drills

1. Trace what `pnpm build` does step by step from `package.json:15`. For each step, name the artifact and who consumes it.
2. Open `rollup.tracker.config.js`. What plugins turn `src/tracker/index.js` into `public/script.js`, and which one fills the `__COLLECT_*__` placeholders?
3. You change the collect endpoint path. List every build step that must rerun for a deployed tracker to use it.

## Interview angle

Prepares you for (see [`../08-interview-prep/`](../08-interview-prep/README.md)):
- "Walk me through your build." → the chain, multi-artifact.
- "ESM vs CJS." → `"type":"module"`, interop friction.
- "Why a different bundler for one part of the app?" → Rollup for the tracker.
- "How do you lint/format?" → Biome consolidation tradeoff.
