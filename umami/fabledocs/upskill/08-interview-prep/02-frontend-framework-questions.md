# Interview Prep: Frontend & Framework Questions

Twelve cards on React 19, state management, data fetching, rendering, and accessibility — anchored to Umami's real dashboard components where possible. Golden rule: concrete example + tradeoff + failure mode.

---

## Q1: How does React decide what to re-render, and how would you find an unnecessary re-render?

**Round:** frontend. **Tests:** render model, profiling.

**Anchor:** conceptual; Umami's dashboard uses React 19 + the React Compiler ([`package.json:101,136`](../../../package.json)).

**Junior:** "When state changes." **Mid adds:** a component re-renders when its state/props change or its parent re-renders; you find waste with the React DevTools Profiler, looking for components rendering when their props are referentially equal. **Senior adds:** React 19's compiler auto-memoizes, so many manual `useMemo`/`useCallback`s are now redundant; the remaining causes of waste are unstable props (new object/array each render) and over-broad store subscriptions. The fix is stabilizing the input, not sprinkling `memo`.

**Follow-ups:** "What changed about memoization in React 19?" "Show me a re-render bug."

**Drill:** Explain how a broad Zustand selector (`useApp(s => s)`) causes re-renders vs. a narrow one (`useApp(s => s.shareToken)`, [`useApi.ts:9`](../../../src/components/hooks/useApi.ts)).

---

## Q2: Where do the three kinds of state live in this app, and why not one store?

**Round:** frontend architecture. **Tests:** state categorization.

**Anchor:** React Query hooks (`components/hooks/queries/*`), Zustand ([`store/app.ts`](../../../src/store/app.ts)), local `useState`.

**Junior:** "In state/context." **Mid adds:** server state (remote data cache) → React Query; global client state (user, theme, share) → Zustand; local UI → `useState`. Umami keeps them separate. **Senior adds:** putting server data in Zustand means re-implementing caching/refetch/invalidation React Query gives you free; putting UI state in a global store causes needless coupling and re-renders. The category dictates the tool. (Umami's `config`/`user` in the store are arguably server data — a fair critique.)

**Follow-ups:** "When would you use Context instead?" "Why not Redux?"

**Drill:** For each of {current theme, website metrics, a form's dirty flag}, name the right tool and cite Umami's choice.

---

## Q3: Explain the `queryKey` in a React Query hook. What breaks if it's incomplete?

**Round:** frontend/data. **Tests:** cache identity.

**Anchor:** [`useWebsiteMetricsQuery.ts:22-31`](../../../src/components/hooks/queries/useWebsiteMetricsQuery.ts).

**Junior:** "It names the query." **Mid adds:** the key *is* the cache identity — every input the result depends on must be in it (websiteId, dates, filters, params). Change a key → refetch; same key → cached. **Senior adds:** an incomplete key is a correctness bug: change an omitted input and React Query serves stale data for the new input — the same failure class as a stale closure. It's not a perf nit, it's wrong data.

**Follow-ups:** "How do you invalidate after a mutation?" "What must be true of objects in a key?" (serializable, stable).

**Drill:** Predict the exact bug if `params.type` were dropped from the metrics key.

---

## Q4: What does `placeholderData: keepPreviousData` do and why use it?

**Round:** frontend/UX. **Tests:** perceived performance.

**Anchor:** [`useWebsiteMetricsQuery.ts:40`](../../../src/components/hooks/queries/useWebsiteMetricsQuery.ts).

**Junior:** "Shows placeholder data." **Mid adds:** while refetching for a new key (e.g. a changed date range), it keeps showing the previous data instead of flashing empty/loading — smoother UX. **Senior adds:** the tradeoff is briefly showing *stale* data labeled as current; fine for dashboards, wrong for something where showing old data is misleading (a balance, a permission state). Pair it with an `isFetching` indicator so users know a refresh is in flight.

**Drill:** When would `keepPreviousData` be the *wrong* choice?

---

## Q5: How does `enabled` change a query's behavior, and why is it here?

**Round:** frontend. **Tests:** conditional fetching.

**Anchor:** `enabled: !!websiteId` ([`useWebsiteMetricsQuery.ts:39`](../../../src/components/hooks/queries/useWebsiteMetricsQuery.ts)).

**Junior:** "Turns the query on/off." **Mid adds:** it defers the fetch until dependencies are ready — here, until `websiteId` exists — avoiding a request with `undefined` in the URL. **Senior adds:** it's the declarative alternative to an imperative "if (id) fetch()" in an effect; it also keeps the query in an `idle` state so loading UI doesn't flash prematurely. Forgetting it causes a wasted/erroring request on first render.

**Drill:** What request would fire without the guard when the page loads before `websiteId` resolves?

---

## Q6: Server Components vs. Client Components — how do you decide?

**Round:** frontend/Next. **Tests:** App Router model.

**Anchor:** Umami's interactive dashboard pieces are `'use client'`; the layout/shell can be server.

**Junior:** "Client for interactivity." **Mid adds:** App Router defaults to Server Components (render on server, no hooks/state, can fetch/DB directly); you opt into Client with `'use client'` where you need state, effects, or browser APIs. Push data-fetching/static content to the server; ship JS only where needed. **Senior adds:** the boundary has a cost — a client component and everything it imports ships to the browser, so you keep the boundary as low in the tree as possible. The tell is `'use client'` at a file's top; misplacing it high in the tree bloats the bundle.

**Follow-ups:** "What can't a Server Component do?" "How does data get from a Server to a Client Component?" (props/serialization).

**Drill:** Given a dashboard page, decide which subtree needs `'use client'` and why.

---

## Q7: This app uses REST hooks, not tRPC/GraphQL. Tradeoffs?

**Round:** frontend architecture. **Tests:** data-layer choices.

**Anchor:** [`useApi.ts`](../../../src/components/hooks/useApi.ts) + ~60 typed hooks.

**Junior:** "REST is standard." **Mid adds:** REST + a thin `useApi` wrapper + React Query gives a plain HTTP API third parties can call, at the cost of hand-writing types on the client (e.g. `WebsiteMetricsData`) with no server→client inference. **Senior adds:** tRPC would give end-to-end type safety but couples client to server and blocks non-JS/third-party consumers; GraphQL solves over/under-fetching but adds a schema/runtime. For a product with a public API (which Umami has), REST is the right call — the manual typing is the accepted cost.

**Drill:** What would Umami gain and lose by switching its dashboard to tRPC?

---

## Q8: How would you handle a form here (submit, validation, error)?

**Round:** frontend. **Tests:** forms + mutations.

**Anchor:** [`LoginForm.tsx`](../../../src/app/login/LoginForm.tsx) + [`useUpdateQuery.ts`](../../../src/components/hooks/queries/useUpdateQuery.ts).

**Junior:** "Handle onSubmit, setState." **Mid adds:** Umami uses a form component with field rules (`rules={{ required }}`), a mutation hook (`useUpdateQuery` → `useMutation`) for the async call, and surfaces `error` from the mutation. On success it sets auth + navigates. **Senior adds:** validation lives client-side for UX *and* server-side (Zod) for trust — never rely on client validation alone. The mutation's `onSuccess`/`error` model separates the async result from field state cleanly.

**Drill:** Trace login submit → mutation → `setClientAuthToken` → redirect ([`LoginForm.tsx:23-30`](../../../src/app/login/LoginForm.tsx)).

---

## Q9: What's a stale closure and where's the React Query analogue?

**Round:** frontend/JS. **Tests:** closures in hooks.

**Anchor:** conceptual; the `queryFn` closes over `startAt`/`filters` ([`useWebsiteMetricsQuery.ts:32-38`](../../../src/components/hooks/queries/useWebsiteMetricsQuery.ts)).

**Mid:** a closure captures variables at definition; a wrong dependency array reuses a stale value. **Senior adds:** in React Query, the defense is that the `queryKey` *is* your dependency list — an omitted input is the stale-closure bug wearing a different hat. Same root cause, two idioms. (Cross-ref [`01-js-ts-node-deep-dive.md`](01-js-ts-node-deep-dive.md) Q10.)

**Drill:** Rewrite the stale-closure risk as a missing-queryKey-input bug and show they're the same.

---

## Q10: How do you make a data table accessible?

**Round:** frontend/a11y. **Tests:** accessibility awareness.

**Anchor:** conceptual; Umami renders many tables (sessions, events) and uses the `@umami/react-zen` component library.

**Junior:** "Add alt text." **Mid adds:** semantic `<table>`/`<th scope>`, keyboard navigability, focus management, `aria-label`s on icon-only controls, sufficient contrast, and not conveying meaning by color alone (relevant to charts). **Senior adds:** the biggest wins are semantic HTML and keyboard operability; test with a screen reader and keyboard-only, not just an audit tool. For data viz, provide a text/table alternative to the chart. A component library helps but doesn't guarantee a11y — you still verify.

**Drill:** List three a11y checks you'd run on the sessions table before shipping.

---

## Q11: Web Vitals — the tracker measures LCP/INP/CLS. What are they and why care?

**Round:** frontend/perf. **Tests:** performance metrics literacy.

**Anchor:** the tracker's performance observers ([`tracker/index.js:251-313`](../../../src/tracker/index.js)); thresholds ([`constants.ts:106-112`](../../../src/lib/constants.ts)).

**Junior:** "Speed metrics." **Mid adds:** LCP (largest contentful paint — loading), INP (interaction to next paint — responsiveness), CLS (cumulative layout shift — visual stability). Good/poor thresholds are in the code. **Senior adds:** they're *field* metrics (real users), which is why Umami captures them via `PerformanceObserver` in the actual browser, not a lab tool. The subtlety: CLS uses session windows and INP uses a 98th-percentile of interactions ([`tracker/index.js:268-313`](../../../src/tracker/index.js)) — averages would mislead. Knowing they're percentile/window-based is a strong signal.

**Drill:** Why does the tracker compute INP as a p98 rather than a max or mean?

---

## Q12: How would you code-split this dashboard?

**Round:** frontend/perf. **Tests:** bundle optimization.

**Anchor:** heavy deps — `react-simple-maps`, `chart.js`, the replay player (`rrweb-player`) — in [`package.json`](../../../package.json).

**Junior:** "Lazy load." **Mid adds:** `import()`/`React.lazy` for heavy, rarely-used views — the world map, the session-replay player — so the initial dashboard bundle doesn't carry them. **Senior adds:** measure first (`next build` chunk sizes); split by route and by heavy component; the replay player and map are prime candidates because they're large and only used in specific views. The tradeoff is a load delay when the view opens, mitigated by prefetch on hover/intent.

**Drill:** Name the two or three heaviest dependencies you'd split first and the view each belongs to.

---

## Drilling this module

Three cards a day, aloud, always ending on a failure mode. Q2, Q3, Q6, Q7 are the ones that escalate into architecture follow-ups — over-prepare those. Pair with [`../02-stack-and-language-mastery/02-framework-mental-models.md`](../02-stack-and-language-mastery/02-framework-mental-models.md).
