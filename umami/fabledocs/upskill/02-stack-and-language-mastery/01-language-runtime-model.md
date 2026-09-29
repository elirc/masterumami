# Language & Runtime Model

## The event loop, from first principles

JavaScript runs on a single thread with an event loop. Synchronous code runs to completion; asynchronous work (I/O, timers) is handed to the host (Node or the browser) and its callback is queued for later. **Microtasks** (resolved promises, `queueMicrotask`) drain completely after each synchronous run, *before* the next **macrotask** (timers, I/O callbacks). Getting this order right is the difference between a mid and a junior answer.

Why it matters in production: on the server, one slow synchronous operation blocks *every* concurrent request, because they share the thread. `await`ing I/O yields the thread so other requests proceed. On the client, blocking the loop freezes the UI.

### Where Umami relies on this

**The collection handler is a chain of `await`s** ([`send/route.ts:150-271`](../../src/app/api/send/route.ts)). Each `await` on a DB call yields the thread back to the loop, so the Node server can interleave other requests while this one waits on Postgres/ClickHouse. The handler *looks* sequential but doesn't monopolize the thread.

**The tracker debounces with a macrotask.** After a SPA navigation, it schedules the track via `setTimeout(track, delayDuration)` ([`tracker/index.js:94`](../../src/tracker/index.js)) — a macrotask that lets the DOM settle (new `document.title`) before reading it.

**`beforeSend` awaits a possibly-async user hook** ([`tracker/index.js:169`](../../src/tracker/index.js)): `payload = await Promise.resolve(callback(type, payload))`. Wrapping in `Promise.resolve` normalizes a sync-or-async return into a thenable — a clean idiom worth stealing.

### Sharp edge in this repo

In `saveEvent`'s relational branch, the event, event-data, and revenue writes are **serial awaits** ([`saveEvent.ts:143-167`](../../src/queries/sql/events/saveEvent.ts)). Some are necessarily serial (revenue needs the event id), but the point to internalize: serial `await`s add their latencies. Independent awaits should be `Promise.all`'d. Recognizing needless serialization is a common perf-review finding.

## Promises, concurrency vs. parallelism

- **Concurrency** = making progress on many things by interleaving (one thread, event loop). **Parallelism** = literally at once (multiple cores/threads). Node gives you concurrency for I/O, not parallelism for CPU.
- `Promise.all` runs awaits *concurrently* (all started, then awaited together) — not in parallel threads, but overlapping their I/O waits.
- A rejected promise with no `.catch`/`try` becomes an unhandled rejection.

### Where Umami relies on this

The **batch endpoint is deliberately sequential** ([`batch/route.ts:21-46`](../../src/app/api/batch/route.ts)): it `await`s each item's `send.POST` in a `for` loop, accumulating a shared `cache`. That's a choice — it trades throughput for a stable cache-token hand-off and ordered error indices. A `Promise.all` would be faster but lose the ordered `cache` accumulation. Being able to argue *why sequential here* is a strong-answer move.

The **Kafka producer swallows rejections** ([`kafka.ts:87-90`](../../src/lib/kafka.ts)) precisely so a downstream failure doesn't reject the request's promise chain — fire-and-forget. The cost (silent loss) is covered in the pattern catalog.

## Node vs. browser: two runtimes, one language

| | Browser (tracker) | Node (server) |
| --- | --- | --- |
| APIs | `window`, `document`, `fetch`, `PerformanceObserver` | `node:crypto`, `fs`, `process.env`, DB clients |
| Secrets | none — everything is public | `APP_SECRET`, DB URLs |
| Modules | bundled by Rollup into one IIFE | ESM (`"type":"module"`) |
| Constraints | tiny size, `keepalive` limits | connection pooling, single thread |

Umami makes this split concrete: `src/lib/crypto.ts` imports `node:crypto` (`:1`) and can *only* run server-side, while `src/tracker/index.js` uses `PerformanceObserver` and `window` and can *only* run in a browser. The `getSession` API the tracker exposes ([`tracker/index.js:391`](../../src/tracker/index.js)) is a browser global; the identity hashing happens on the server where the secret lives.

## Modules & ESM

The repo is ESM (`"type": "module"`). That means `import`/`export`, top-level `await` availability, and `.js` extensions mattering in some contexts. The `@/` alias (see `tsconfig.json`) maps to `src/`. The tracker, by contrast, is compiled by Rollup into a single self-executing function so it can be dropped into any page as one `<script>`.

## Pitfall checklist (JS/Node)

- [ ] Serial `await`s that could be `Promise.all` — scan `saveEvent`, any handler with multiple independent writes.
- [ ] `||` where you meant `??` (a valid `0`/`''`/`false` getting defaulted) — see `pageSize || DEFAULT` (`prisma.ts:286`) vs. nullish cases.
- [ ] Swallowed promise rejections hiding failures — `kafka.ts:87`.
- [ ] Blocking the event loop with sync CPU work in a request (none egregious here, but pbkdf2 in `crypto.ts:16` is sync — fine for login, would be costly per-event).
- [ ] Assuming `process.env` exists in browser code — it doesn't; it's injected at build.
- [ ] Reading `document.title` before the DOM updates on SPA nav — why the tracker `setTimeout`s.

## Drills

1. **Event-loop order:** In `beforeSend` handling (`tracker/index.js:166-172`), a synchronous callback and an async one are both supported. Explain the execution order of the code after `await Promise.resolve(callback(...))` for each case.
2. **Serial vs parallel:** Rewrite `saveEvent`'s relational branch to run event-data and revenue writes concurrently *where correct*. Which write must still wait, and why?
3. **Runtime boundary:** List three functions in `src/lib/` that could never run in the tracker, and say which import gives it away.

## Interview angle

This file prepares you for (see [`../08-interview-prep/01-js-ts-node-deep-dive.md`](../08-interview-prep/01-js-ts-node-deep-dive.md)):
- "Walk me through the event loop during an async request handler." → Q2.
- "Concurrency vs. parallelism in Node." → Q2, drill 2.
- "Why does the tracker use `keepalive`?" → Q1.
- "`??` vs `||`." → Q11.
- "Browser vs. Node constraints for the same language." → drill 3.
