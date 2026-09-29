# 05 — Quality Engineering

The skills that separate "it works on my machine" from "it works in production and I can prove it": testing, debugging, performance, security, and observability. Each file is grounded in Umami's actual tooling and code.

Files:

1. [`01-testing-strategy.md`](01-testing-strategy.md) — the test layers here, what belongs at each, flake prevention.
2. [`02-writing-tests-here.md`](02-writing-tests-here.md) — concrete test recipes using this repo's helpers, with commands.
3. [`03-systematic-debugging.md`](03-systematic-debugging.md) — the reproduce→narrow→fix→cover method on 5 real scenarios.
4. [`04-performance-thinking.md`](04-performance-thinking.md) — the performance domains here and how to find hotspots.
5. [`05-security-checklist.md`](05-security-checklist.md) — authz/IDOR/injection/XSS/CSRF/secrets mapped to real code, plus a pre-merge checklist.
6. [`06-observability-and-operations.md`](06-observability-and-operations.md) — logging, health, deploy/rollback, and "how would I know it broke?"

Start with `01` for the map, then `02` to actually write one. `03` is the highest-value interview prep in this module.
