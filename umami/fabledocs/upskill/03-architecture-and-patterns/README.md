# 03 — Architecture and Patterns

Where module 01 mapped *what's where*, this module explains *why it's shaped that way* — the boundaries, the data model, the security model, the reliability model — and turns recurring shapes into a pattern vocabulary you can carry to any codebase.

Files:

1. [`01-boundaries-and-layers.md`](01-boundaries-and-layers.md) — the layers, what each owns, and where boundaries leak.
2. [`02-data-model-and-persistence.md`](02-data-model-and-persistence.md) — the schema, indexes, the Postgres/ClickHouse split, transactions, and safe schema change.
3. [`03-validation-auth-and-permissions.md`](03-validation-auth-and-permissions.md) — the full validation/authN/authZ picture, multi-tenancy, IDOR.
4. [`04-side-effects-async-and-reliability.md`](04-side-effects-async-and-reliability.md) — every side effect, idempotency, retries, the fire-and-forget gap.
5. [`05-pattern-catalog.md`](05-pattern-catalog.md) — **16 pattern cards** for recognition and interviews.
6. [`06-architecture-critique.md`](06-architecture-critique.md) — strongest choices, real risks, and "what I'd change owning this for 3 months." Doubles as system-design interview material.

Mid-level and senior readers: start with `06-architecture-critique.md`, then use the others as evidence. Juniors: read 01 → 05 in order.
