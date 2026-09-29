# 02 — Stack and Language Mastery

Umami's stack is the exact stack a mid-level fullstack JS interview targets: TypeScript on Node, React 19 on the client, Prisma + raw SQL for data. This module builds the *mental models* — from first principles → where Umami uses each well → where it has sharp edges → a pitfall checklist → drills on real files.

Files:

1. [`01-language-runtime-model.md`](01-language-runtime-model.md) — the JS event loop, promises/microtasks, and Node vs. browser, seen through the tracker and the collection path.
2. [`02-framework-mental-models.md`](02-framework-mental-models.md) — React 19 render/commit, hooks, effects, React Query, Server vs. Client Components, Zustand.
3. [`03-type-system-and-contracts.md`](03-type-system-and-contracts.md) — TS inference, unions, narrowing, `satisfies`, Zod as the runtime edge of the type system.
4. [`04-tooling-and-build-system.md`](04-tooling-and-build-system.md) — ESM, Turbopack, Rollup for the tracker, Biome, and why the build is a chain.

Each file ends with an **Interview angle** section listing the questions it prepares you for, cross-linked into [`../08-interview-prep/`](../08-interview-prep/README.md). Read those even if you skip the drills.
