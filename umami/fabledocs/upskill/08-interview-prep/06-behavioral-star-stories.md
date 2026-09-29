# Interview Prep: Behavioral STAR Stories

Ten STAR worksheets you can build from *actually doing* the work in this curriculum. STAR = Situation, Task, Action, Result. The mid-level signal isn't a dramatic story — it's a *specific* one that ends in impact and shows judgment (a tradeoff weighed, a risk reduced, a person helped).

If you complete tickets/projects from [`../06-contribution-practice/`](../06-contribution-practice/README.md), these become *your* stories with real evidence. Even studying the repo deeply gives you the "learned a complex codebase" story.

Each worksheet: prompts it answers → source → the four beats → evidence → the resume-bullet version. Keep the spoken version under 2 minutes, concrete, ending on impact.

---

## Story 1: Learning a complex codebase fast

**Answers:** "Walk me through a time you got up to speed on an unfamiliar codebase." "How do you approach a new system?"

**Source:** studying Umami (this curriculum).

- **S:** I was handed a large analytics codebase — a Next.js app with a vanilla-JS tracker, dual Postgres/ClickHouse persistence, and hand-written SQL.
- **T:** Understand it well enough to reason about and change it safely.
- **A:** I mapped it before diving in — identified the two flows that define it (collection and metrics reads), traced each end to end with file anchors, and built a reading order from the hot path outward instead of reading randomly.
- **R:** Within days I could explain the write path, name its failure modes (silent Kafka loss, a latent injection surface), and make safe changes. I turned that map into onboarding notes.

**Evidence:** the traced flows, the risk register.
**Resume bullet:** *Reverse-engineered a production analytics system (Next.js, ClickHouse, hand-written SQL) into a documented map of flows and risks.*

---

## Story 2: Improving reliability (the outbox)

**Answers:** "Tell me about a time you improved system reliability." "A time you prevented a production issue."

**Source:** Project S1.

- **S:** The collection path published events to Kafka fire-and-forget; a broker outage dropped events while returning success — silent data loss for the product's core data.
- **T:** Make delivery not lose data.
- **A:** I designed an outbox: durably persist on accept, relay to Kafka with retries. I handled the duplicate consequence with the system's deterministic ids and `on conflict do nothing`, ran it in shadow mode, and added a reconciliation metric to verify no loss during cutover.
- **R:** At-least-once delivery with idempotent dedup; a previously invisible failure became measurable.

**Evidence:** the RFC/design doc, the reconciliation metric.
**Resume bullet:** *Converted a fire-and-forget event pipeline to at-least-once delivery via an outbox + relay, eliminating silent data loss.*

---

## Story 3: Finding and closing a security gap

**Answers:** "A time you found a security issue." "A time you improved security."

**Source:** the `orderBy` injection surface / IDOR guardrail (Ticket 6, Project S2).

- **S:** I noticed the raw-SQL pagination interpolated `orderBy` directly — a latent injection surface — and that authorization was a manual per-route check easy to forget.
- **T:** Reduce the risk without overclaiming a live vuln.
- **A:** I confirmed the injection wasn't currently reachable (no route fed user input), documented the invariant that keeps it safe, and proposed an allow-list. For authz, I proposed a wrapper/lint so a forgotten check couldn't ship.
- **R:** A latent vulnerability documented and de-risked; a plan to make an entire vuln class unrepresentable.

**Evidence:** risk register entries, the guardrail design.
**Resume bullet:** *Identified a latent SQL-injection surface and an IDOR-prone authorization pattern; proposed allow-listing and an enforced authorization wrapper.*

---

## Story 4: A technical tradeoff you weighed

**Answers:** "Tell me about a hard technical decision." "A time you weighed tradeoffs."

**Source:** Kata 4 (don't over-abstract the dual SQL branches).

- **S:** The codebase duplicated near-parallel Postgres and ClickHouse query implementations that could drift.
- **T:** Reduce drift without a leaky abstraction.
- **A:** I resisted forcing a shared abstraction — the two dialects genuinely differ (ClickHouse denormalizes sessions onto events). Instead I proposed a parity test comparing results on a seeded dataset, catching drift without coupling.
- **R:** Drift protection with the two implementations kept honest and readable.

**Evidence:** the kata write-up.
**Resume bullet:** *Chose a parity-test guard over a forced abstraction to prevent dual-backend query drift — reducing risk without coupling.*

---

## Story 5: Fixing a subtle bug (cache staleness)

**Answers:** "The hardest bug you've debugged." "A time you found a root cause others missed."

**Source:** Debugging Scenario 5 / Ticket M4.

- **S:** Disabling session replay didn't take effect for up to a day on one path but was instant on another.
- **T:** Explain and fix the inconsistency.
- **A:** I traced both paths to their website reads — one used a 24h Redis cache, one deliberately bypassed it. The root cause was missing cache invalidation on update, not the bypass. I proposed invalidating on write rather than bypassing on read.
- **R:** Consistent, immediate behavior; a general fix instead of a per-path patch.

**Evidence:** the trace, the fix design.
**Resume bullet:** *Root-caused a cache-staleness bug to missing write-time invalidation and fixed it generally rather than per-endpoint.*

---

## Story 6: Making the invisible observable

**Answers:** "A time you improved observability/monitoring." "How did you know something was wrong in prod?"

**Source:** Project S3 / Ticket M12.

- **S:** The system had no metrics — a failure that dropped events was invisible until customers complained.
- **T:** Make failures detectable.
- **A:** I added accepted/persisted/error counters and an accepted-vs-persisted reconciliation metric, plus correlation ids on error responses, keeping the hot path cheap with in-memory counters.
- **R:** Silent data loss became a graphable, alertable signal.

**Resume bullet:** *Added reconciliation metrics and correlation-id logging that surfaced previously silent event-loss failures.*

---

## Story 7: Giving difficult code-review feedback

**Answers:** "A time you disagreed with a teammate's code." "A time you gave hard feedback."

**Source:** Review Katas (e.g. the "typo fix" that would break referrer tracking).

- **S:** A teammate's PR "fixed" `referer` to `referrer` — but `Referer` is the correct (spec-misspelled) HTTP header; the change would silently break referrer tracking.
- **T:** Stop the regression without discouraging them.
- **A:** I explained the HTTP spec quirk, cited the line, and suggested a comment documenting the intentional spelling so it wouldn't recur — framed as "careful, this one's a famous gotcha," not "you're wrong."
- **R:** Regression avoided; the teammate learned the quirk; a comment now prevents the next person from repeating it.

**Resume bullet:** *Caught a plausible-looking "typo fix" that would have broken referrer tracking, and added documentation to prevent recurrence.*

---

## Story 8: Scoping down / resisting over-engineering

**Answers:** "A time you kept something simple." "A time you pushed back on scope."

**Source:** Kata 3 (splitting `lib/prisma.ts`) / general contribution discipline.

- **S:** A 437-line module mixing several concerns tempted a big refactor.
- **T:** Improve it without a risky rewrite.
- **A:** I proposed a cohesive split (engine / filters / pagination / client) sequenced to keep every step green, and explicitly argued *against* over-splitting — small enough to review, not so granular it fragmented the logic.
- **R:** A safer, reviewable refactor plan; the public export stayed stable.

**Resume bullet:** *Planned an incremental module split that improved cohesion while keeping every step shippable and the public API stable.*

---

## Story 9: Handling ambiguity

**Answers:** "A time you worked with unclear requirements." "A time you had to make a call with incomplete info."

**Source:** the "visitor counts reset monthly" investigation (Scenario 1).

- **S:** A reported "bug": returning-visitor counts reset every month.
- **T:** Determine if it was a bug and what to do.
- **A:** Rather than fixing the symptom, I separated whether events or only visitors dropped, traced the identity derivation to a deliberate monthly salt rotation for privacy, and reframed the ticket from "bug" to "product decision about the identity window."
- **R:** Avoided breaking a privacy feature; gave the PM the real tradeoff to decide on.

**Resume bullet:** *Reframed a reported "bug" as an intentional privacy tradeoff by tracing the mechanism, preventing a regression of a core feature.*

---

## Story 10: Teaching / leveling up others

**Answers:** "A time you helped a teammate grow." "A time you shared knowledge."

**Source:** producing this curriculum / onboarding notes.

- **S:** New engineers struggled to onboard onto a dense analytics codebase.
- **T:** Make it learnable.
- **A:** I built a reading order, traced the key flows with anchors, and wrote drills and a risk register so others could reason about the system, not just navigate it.
- **R:** Faster onboarding and a shared vocabulary for the system's tradeoffs.

**Resume bullet:** *Authored onboarding docs (flow traces, reading order, risk register) that accelerated new-engineer ramp-up on a complex codebase.*

---

## Using these

- **Fill the beats with specifics** — real files, real numbers where you have them. Vagueness is the enemy.
- **Map to prompts:** conflict → Story 7; ambiguity → Story 9; mistake → adapt one where you initially took the wrong path; tradeoff → Story 4; leadership → Story 10.
- **Rehearse to under 2 minutes**, ending on impact. Time yourself.
- **The meta-move:** having *done* the tickets/projects makes these true. Do two or three from module 06 and these stories stop being hypothetical.

## Rehearsal check (per story)

- [ ] Under 2 minutes spoken?
- [ ] Concrete (names a file, a decision, a number)?
- [ ] Shows a tradeoff or judgment, not just effort?
- [ ] Ends on measurable impact?
