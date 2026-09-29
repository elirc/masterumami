# Two-Week Cram Plan

A day-by-day plan for someone with an interview in two weeks who has this repo as their study material. It weaves the question cards, one mock system design, two timed debugging rounds, story rehearsal, and teach-backs, with checkpoints at day 7 and day 14. Assume ~2 focused hours/day; scale down by cutting drills, not concepts.

The spine: **learn the system deeply enough to answer any concept question with a concrete Umami example + tradeoff + failure mode.**

## Week 1 — Build the foundation and the vocabulary

### Day 1 — Orient
- Read [`../00-fast-track.md`](../00-fast-track.md) end to end.
- Trace Flow 1 (collection) in [`../01-codebase-cartography/05-key-flows.md`](../01-codebase-cartography/05-key-flows.md) with the files open.
- **Teach-back:** explain the collection flow aloud in 2 minutes.

### Day 2 — The read path & the system map
- Trace Flow 2 (metrics read) and Flow 3 (login/tokens).
- Read [`../01-codebase-cartography/01-system-map.md`](../01-codebase-cartography/01-system-map.md).
- **Cards:** JS/TS deep-dive Q1–Q4 ([`01-js-ts-node-deep-dive.md`](01-js-ts-node-deep-dive.md)), aloud.

### Day 3 — Patterns
- Read [`../03-architecture-and-patterns/05-pattern-catalog.md`](../03-architecture-and-patterns/05-pattern-catalog.md); for each of the top 8 patterns, say the problem/tradeoff aloud.
- **Cards:** JS/TS Q5–Q9.

### Day 4 — Data & API modeling
- Read [`../03-architecture-and-patterns/02-data-model-and-persistence.md`](../03-architecture-and-patterns/02-data-model-and-persistence.md).
- **Cards:** API & data Q1–Q6 ([`03-api-and-data-modeling-questions.md`](03-api-and-data-modeling-questions.md)).

### Day 5 — Auth, security, frontend
- Read [`../03-architecture-and-patterns/03-validation-auth-and-permissions.md`](../03-architecture-and-patterns/03-validation-auth-and-permissions.md) and [`../05-quality-engineering/05-security-checklist.md`](../05-quality-engineering/05-security-checklist.md).
- **Cards:** frontend Q1–Q6 ([`02-frontend-framework-questions.md`](02-frontend-framework-questions.md)).

### Day 6 — First system-design pass
- Read [`04-system-design-from-this-repo.md`](04-system-design-from-this-repo.md) fully.
- Whiteboard "design a web analytics platform" from the prompt alone (20 min), then compare to what Umami did. Note your gaps.

### Day 7 — Checkpoint + rest
- **Self-assessment (Day 7):**
  - [ ] Can I trace collection and metrics-read flows without notes?
  - [ ] Can I name 8 patterns with a tradeoff each?
  - [ ] Can I explain cookieless identity, dual persistence, and the IDOR guard?
  - [ ] Did I answer ≥15 cards aloud ending on a failure mode?
- Reread the [`../09-reference/learning-rubrics.md`](../09-reference/learning-rubrics.md) and honestly place yourself. Light day otherwise.

## Week 2 — Simulate and sharpen

### Day 8 — Debugging round 1 (timed)
- Do Debugging Simulation 1 (silent event loss) and 2 (cross-tenant leak) from [`05-debugging-and-code-review-rounds.md`](05-debugging-and-code-review-rounds.md), **10 min each, narrating aloud, recorded.**
- Listen back; note filler and skipped steps.
- **Cards:** JS/TS Q10–Q14.

### Day 9 — Review round + remaining cards
- Do Review Simulations 1 and 2 (timed, aloud).
- **Cards:** API & data Q7–Q12.

### Day 10 — Frontend depth + reliability
- **Cards:** frontend Q7–Q12.
- Read [`../03-architecture-and-patterns/04-side-effects-async-and-reliability.md`](../03-architecture-and-patterns/04-side-effects-async-and-reliability.md) — the outbox story is a top talking point.

### Day 11 — Second system-design pass + a variation
- Redo the system-design walkthrough, this time hitting a variation prompt ("10× traffic" or "make it real-time"). 25 min, aloud.
- Read [`../03-architecture-and-patterns/06-architecture-critique.md`](../03-architecture-and-patterns/06-architecture-critique.md) — the "what I'd change" framing is the senior-signal closer.

### Day 12 — Behavioral
- Fill 5 STAR worksheets in [`06-behavioral-star-stories.md`](06-behavioral-star-stories.md) with your specifics (do a ticket or two from module 06 if you have time, to make them real).
- Rehearse each under 2 minutes, ending on impact.

### Day 13 — Debugging round 2 + mixed review
- Debugging Simulation 3 (slow endpoint), timed.
- Re-answer any cards you fumbled. Do 2 annotation drills ([`../04-code-reading-gym/01-annotation-drills.md`](../04-code-reading-gym/01-annotation-drills.md)) for code-reading warmth.

### Day 14 — Checkpoint + full dress rehearsal
- **Self-assessment (Day 14):**
  - [ ] Can I whiteboard the analytics design and defend each choice with a tradeoff?
  - [ ] Can I run a debugging round narrating, splitting the space first, ending with a regression test?
  - [ ] Can I review a diff, grade severity, and phrase feedback kindly?
  - [ ] Do I have 5 concrete STAR stories under 2 minutes each?
  - [ ] For any concept, can I reach for a Umami example + tradeoff + failure mode?
- Do one full mock: 1 system-design + 1 debugging + 2 behavioral, back to back, timed. Rest before the interview.

## If you have less than two weeks

- **1 week:** Days 1–3, 6, 8, 12, 14. Skip the second passes.
- **3 days:** Fast-track + Flow 1 + system-design walkthrough + JS/TS cards Q1–Q6 + 3 STAR stories. That alone puts a concrete system in your back pocket.

## The single most important habit

Every time you answer *anything*, end with a failure mode or tradeoff. "…and where that breaks is…" / "…the cost of that is…". That one habit is the difference between a junior and a mid-level answer, and this whole repo exists to give you the material for it. Go get the offer.
