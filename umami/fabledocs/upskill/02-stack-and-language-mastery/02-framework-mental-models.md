# Framework Mental Models

## React, from first principles

React is a function of state: `UI = f(state)`. When state changes, React re-runs your component function to produce a new virtual tree (**render**), diffs it against the last one, and applies the minimal DOM changes (**commit**). Render must be pure — no side effects — because React may run it multiple times or discard the result. Side effects belong in **effects** (`useEffect`), which run *after* commit.

Why this matters: most React bugs come from treating render as if it runs once, or from putting effects/derivations in the wrong place. A mid-level engineer can say *where* a piece of logic belongs (render vs. effect vs. event handler vs. server) and why.

## React 19 + the React Compiler

Umami uses React 19 with `babel-plugin-react-compiler` ([`package.json:101,136`](../../package.json)). The compiler auto-memoizes — it inserts the equivalent of `useMemo`/`useCallback` for you, so hand-memoization is less necessary than in older React. The mental model shift: you write straightforward components and the compiler handles referential stability. In an interview, note that this reduces (not eliminates) the need for manual memoization and changes the "why is this re-rendering?" conversation.

## State: three kinds, three tools

Umami cleanly separates three categories of state — a distinction interviewers probe:

| Kind | Example | Tool in Umami | Anchor |
| --- | --- | --- | --- |
| **Server state** (cache of remote data) | website metrics, sessions | **React Query** | `components/hooks/queries/*` |
| **Global client state** (app-wide UI) | current user, theme, timezone, active share | **Zustand** | `src/store/app.ts` |
| **Local component state** | a form field, a toggle | `useState` | throughout components |

The lesson: don't put server data in Zustand or `useState` — you'd be hand-rolling caching, refetching, and invalidation that React Query already does. And don't put ephemeral UI state in a global store. Umami's split is textbook; you can cite it.

### React Query mental model

A `useQuery` is a **subscription to a cached async resource keyed by inputs**. The `queryKey` is the cache identity: same key → same cached entry; changed key → refetch. Everything the result depends on must be in the key.

Real example — [`useWebsiteMetricsQuery.ts:21-42`](../../src/components/hooks/queries/useWebsiteMetricsQuery.ts):
- `queryKey` = `['websites:metrics', { websiteId, startAt, endAt, ...filters, ...params }]` — every input is present.
- `enabled: !!websiteId` — defer the fetch until the id exists (avoids a request with `undefined`).
- `placeholderData: keepPreviousData` — show the last data while refetching, so changing a filter doesn't flash empty.

Mutations follow the sibling pattern ([`useUpdateQuery.ts`](../../src/components/hooks/queries/useUpdateQuery.ts)): a `useMutation` calling `post`, plus a `touch()` to signal "data changed."

### Zustand mental model

A minimal global store: `create(() => initialState)` and setter functions that call `store.setState` ([`store/app.ts:25-52`](../../src/store/app.ts)). Components subscribe via `useApp(selector)`; a narrow selector (like `useApi`'s `state => state.shareToken`, [`useApi.ts:9`](../../src/components/hooks/useApi.ts)) means the component re-renders only when that slice changes. The lesson: **select narrowly** or you re-render on unrelated store updates.

## Effects: the part juniors overuse

An effect synchronizes your component with something *outside* React (the DOM, a subscription, a non-React store). If you can compute a value during render, you don't need an effect. If you can do it in an event handler (like a click), you don't need an effect. Umami's data fetching lives in React Query (not raw `useEffect`), which is the modern "don't fetch in an effect" guidance in practice.

Common effect bugs to recognize:
- **Missing/incorrect dependency array** → stale closures (a captured old value). The React Query analogue is an incomplete `queryKey` (see [`../08-interview-prep/01-js-ts-node-deep-dive.md`](../08-interview-prep/01-js-ts-node-deep-dive.md) Q10).
- **Effect that should be an event handler** → runs on every render instead of on the action.

## Server vs. Client Components (App Router)

Next.js App Router defaults components to **Server Components** (render on the server, no hooks/state, can touch the DB) unless marked `'use client'`. Umami's interactive dashboard pieces are client components (they use hooks like `useApi`, `useApp`). The mental model: push data-fetching and static rendering to the server; opt into client only where you need interactivity/state. In an interview, the tell is `'use client'` at the top of a file — it marks the boundary where JS ships to the browser.

## Data fetching layer

Umami does **not** use tRPC or GraphQL — it's REST over a thin `useApi` wrapper ([`useApi.ts`](../../src/components/hooks/useApi.ts)) that injects the bearer token and share headers, wrapped by ~60 typed React Query hooks. The tradeoff vs. tRPC: no end-to-end type inference from server to client (you type the hooks by hand, e.g. `WebsiteMetricsData`), but zero coupling and a plain HTTP API third parties can call. Being able to compare REST-hooks vs. tRPC is a good frontend-architecture answer.

## Pitfall checklist (React)

- [ ] Server data in `useState`/Zustand instead of React Query.
- [ ] Missing input in a `queryKey` → stale data for new inputs.
- [ ] Broad Zustand selectors causing extra re-renders.
- [ ] Logic in an effect that belongs in render or an event handler.
- [ ] Forgetting `enabled` guard → firing queries with undefined params.
- [ ] Unstable objects in a `queryKey` (non-serializable/new-every-render).

## Drills

1. Open three hooks in `components/hooks/queries/`. For each, list every input in the `queryKey` and confirm the `queryFn` uses exactly those. Find any mismatch.
2. In `store/app.ts`, which state slices are server data that arguably belong in React Query instead? (`config`, `user` are candidates — argue both sides.)
3. Find one client component (`'use client'`) and one that isn't. What forced the client boundary?

## Interview angle

Prepares you for (see [`../08-interview-prep/02-frontend-framework-questions.md`](../08-interview-prep/02-frontend-framework-questions.md)):
- "Render vs. commit; where do side effects go?"
- "Three kinds of state and which tool for each."
- "How does React Query cache and invalidate?" → the metrics hook.
- "Server vs. Client Components — how do you decide?"
- "REST hooks vs. tRPC/GraphQL — tradeoffs."
