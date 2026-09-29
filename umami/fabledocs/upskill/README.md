# Umami Upskill Curriculum

A training lab built on the **Umami** codebase (`umami` v3.1.0 — `package.json:3`), a privacy-focused, self-hostable web analytics platform. This is not documentation *for* Umami; it is a curriculum that uses Umami as a real, production-grade specimen to take a junior fullstack JS engineer to solid mid-level and prepare them for mid-level fullstack interviews.

Every page teaches two things at once:

- **This repo**: where things live, how its real flows work, what its authors decided.
- **The transferable skill**: why the pattern exists, its alternatives and failure modes, and how to recognize and explain it in *any* codebase — and in an interview.

## Why Umami is a good teaching specimen

1. It is a **real product**, not a toy. Analytics collection is a genuine high-throughput write path with genuine tradeoffs (privacy vs. accuracy, cost vs. detail).
2. It has a **clean layered shape**: Zod validation → auth → permission → query layer → dispatch to Postgres *or* ClickHouse. You can see each boundary.
3. It exercises the exact topics interviewers probe: async data flow, an event loop hot path, SQL and schema design, caching, authz/IDOR, multi-tenancy, and a real system-design story ("design an analytics platform").
4. It has a **polymorphic persistence layer** — the same logical write targets a relational DB or a columnar analytics DB — which is a rare and impressive thing for a junior to be able to explain.
5. It ships a **vanilla-JS tracker** and a **React 19 dashboard** in one repo, so you get both the client edge and the server core.
6. It has **honest sharp edges** (fire-and-forget Kafka, a latent `orderBy` injection surface, no app-level rate limiting) that make excellent "what would a senior inspect first?" material.

Read the repo's identity in one paragraph: *Umami receives anonymous events from a tiny tracker script over an unauthenticated `POST /api/send`, derives a cookieless visitor identity by hashing IP + user-agent + a rotating salt, and writes each event to either Postgres (via Prisma) or ClickHouse (optionally through Kafka). A Next.js 16 App Router dashboard, guarded by bearer tokens and per-entity permission checks, reads aggregated metrics back through a hand-written parameterized SQL layer. Teams, shares, reports, funnels, session replay, and UTM attribution are layered on top.*

## Learning tracks

| Module | What it builds |
| --- | --- |
| `01-codebase-cartography/` | A map of the repo: system shape, reading order, domain glossary, tooling, and 6 traced end-to-end flows. |
| `02-stack-and-language-mastery/` | JS/TS/Node/React 19 mental models, anchored to where Umami uses (and stresses) them. |
| `03-architecture-and-patterns/` | Boundaries, the data model, validation/authz, async & reliability, a pattern catalog, and an architecture critique. |
| `04-code-reading-gym/` | Annotation drills, trace tables, bad-vs-better contrasts, review katas — active practice. |
| `05-quality-engineering/` | Testing strategy & recipes, systematic debugging, performance, security checklist, observability. |
| `06-contribution-practice/` | Graded tickets: 20+ junior, 12 mid-level, 6 senior projects, refactor katas. |
| `07-career-and-collaboration/` | Code review mindset, PR/RFC writing, maintainer communication. |
| `08-interview-prep/` | **First-class module.** 50+ question cards (≥60% anchored to this repo), a full system-design walkthrough, STAR stories, and a two-week cram plan. |
| `09-reference/` | Command cheatsheet, risk register, learning rubrics, and the running verification log. |

## How to use this over different time budgets

- **One weekend**: `00-fast-track.md` end to end. Trace the collection flow, run one test, make one small change.
- **Two weeks (you have an interview)**: `00-fast-track.md`, then `08-interview-prep/07-two-week-cram-plan.md` as your spine. It pulls the right pages from every module day by day.
- **Eight weeks (deep upskilling)**: Modules 01 → 08 in order, doing every drill and at least three tickets from `06-contribution-practice/`.
- **Ongoing contribution**: Live in `06-contribution-practice/` and `07-career-and-collaboration/`; use `09-reference/risk-register.md` to find real work.

## Recommended paths by who you are

- **Brand-new junior**: `00-fast-track.md` → `01-codebase-cartography/` → `02-stack-and-language-mastery/` → `04-code-reading-gym/`. Don't rush to tickets.
- **Junior with React/Node familiarity**: skim `01`, go deep on `03-architecture-and-patterns/` and `05-quality-engineering/`, then take mid-level tickets.
- **Mid-level new to the repo**: `01-codebase-cartography/05-key-flows.md` → `03-architecture-and-patterns/06-architecture-critique.md` → `09-reference/risk-register.md`.
- **Senior doing architecture review**: `03-.../06-architecture-critique.md` and `09-reference/risk-register.md` first; the flows and pattern catalog are your evidence base.
- **Candidate with an interview in two weeks**: go straight to [`08-interview-prep/07-two-week-cram-plan.md`](08-interview-prep/07-two-week-cram-plan.md).

## Conventions used throughout

- **File anchors.** Every claim about the code points at `path:line` or a `[link](../relative/path.ts#L10-L20)`. If a line number and the code disagree, trust the code and tell us — line numbers drift.
- **Fake code is labelled.** Any snippet not copied from the repo begins with `// Illustrative fake code: not from this repo`. Real excerpts are anchored.
- **Drills and self-grading.** Pages end with a task and a Basic / Solid / Strong rubric. Do the drill before reading the answer.
- **Uncertainty is labelled.** Suspicions are marked "possible risk" or "investigate," never asserted as bugs. Confirmed facts are separated from hypotheses — see [`09-reference/verification-log.md`](09-reference/verification-log.md).
- **Senior vocabulary, defined in place.** Words like *invariant*, *boundary*, *idempotency*, *blast radius*, *authorization*, *consistency* are used where they truly apply and defined the first time near real code. These are also interview vocabulary; the interview module flags them.

## The mindset ladder

- **Junior** asks: *"How do I make it work?"*
- **Mid-level** asks: *"Is this the right pattern, and how does it fail?"*
- **Senior** asks: *"What does this commit us to, who pays the cost, and how do I shrink the blast radius?"*

Interviews for mid-level roles are testing whether you have started asking the **second** question and can gesture at the **third**. This whole curriculum is organized to move you up that ladder using code you can point at.

Start with [`00-fast-track.md`](00-fast-track.md).
