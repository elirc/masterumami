# Maintainer Communication

How you ask questions, report bugs, and disagree marks you as junior or mid-level faster than your code does. The through-line: **do your own thinking first, then communicate precisely.**

## Asking for help without outsourcing your thinking

The junior move: "It doesn't work, what do I do?" The mid-level move: "Here's what I'm seeing, what I've tried, my current hypothesis, and the specific thing I'm stuck on."

A good help request has done the work up to the wall:

```
I'm adding a filter for X. Events with X aren't showing up when I filter.
- I confirmed the value reaches the route (logged `query`, it's there).
- I traced it to `parseFilters` (prisma.ts:232) and I think the issue is
  X isn't in FILTER_COLUMNS, so getFilterQuery skips it (prisma.ts:120).
- I added it to FILTER_COLUMNS and the Postgres path works, but the
  ClickHouse path still ignores it.
Question: is there a second place ClickHouse filters need registering,
or am I missing a mapping? Anchors above.
```

This respects the maintainer's time, proves you're capable, and gets a precise answer. The `LOG_QUERY`/`DEBUG=umami:*` tools ([observability](../05-quality-engineering/06-observability-and-operations.md)) are how you gather this evidence before asking.

## Writing a repro report (bugs)

`CONTRIBUTING.md` asks for repro steps, expected vs. actual, and environment. Make the repro **minimal** — strip everything not needed to trigger it.

```markdown
**Environment:** Umami v3.1.0, Postgres 15, Chrome 120, Redis off.

**Steps to reproduce:**
1. Toggle `replayEnabled` off on a website.
2. Immediately send an event on the collection path.
3. Observe replay still behaves as enabled for ~24h.

**Expected:** disabling replay takes effect immediately.
**Actual:** takes up to 24h on the collection path (instant on /record).

**Notes:** looks related to the Redis website cache TTL (load.ts:10) vs.
the deliberate bypass in record/route.ts:56. Not confirmed.
```

A report like this is often *half the fix* — the maintainer can act on it immediately. Label a hypothesis as a hypothesis ("looks related… not confirmed"), never as a fact.

## Proposing a feature

Lead with the **use case**, not your solution (this is explicit in `CONTRIBUTING.md`):

> **Use case:** operators can't tell when events are accepted but not persisted (the Kafka-drop gap), so silent data loss is invisible.
> **Proposed:** an accepted-vs-persisted reconciliation metric. (Details in an RFC if there's interest.)

Leading with the solution invites bikeshedding; leading with the problem invites agreement on *what* before *how*.

## Respectful disagreement

You will sometimes think a maintainer is wrong. How you handle it matters more than being right.

- **Assume they have context you don't.** Ask before asserting: "What's the reasoning behind X? I was expecting Y because…"
- **Argue the technical merits, with evidence.** "Benchmarks show…" / "This anchor suggests…" beats "I think it's better."
- **Disagree and commit.** If the decision goes against you after a fair hearing, support it. Re-litigating a settled call burns goodwill.
- **Pick the hill.** Push hard on correctness/security; let go of style preferences.

Example: *"I hear you on keeping it simple. My worry is the fire-and-forget send loses events silently (kafka.ts:87). Could we at least log a metric so we'd know? Happy to defer the full outbox."* — a disagreement that offers a smaller step.

## Templates

**Asking a question:**
> Context: [what you're doing]. I traced it to [anchor] and think [hypothesis]. I tried [X, Y]. Specific question: [one clear question].

**Reporting a bug:** (see repro template above) — environment, steps, expected, actual, hypothesis-labelled notes.

**Proposing a feature:**
> Use case: [problem]. Impact: [who's affected]. Proposed direction: [brief]. Want me to write an RFC?

**Responding to review feedback:**
> Good catch on [X] — fixed in [commit]. On [Y], I chose [approach] because [reason]; happy to change if you'd prefer [alternative]. Pushed back gently on [Z] with [evidence] — let me know.

## The meta-principle

Every one of these is the same skill: **respect people's time by doing your homework and being precise.** A maintainer who gets a well-researched question, a minimal repro, or a use-case-first proposal will want to work with you. That reputation is career capital.

## Drill

Take Debugging Scenario 5 (the replay-staleness bug, [`../05-quality-engineering/03-systematic-debugging.md`](../05-quality-engineering/03-systematic-debugging.md)) and write it up as (a) a help request if you were stuck, and (b) a bug report if you'd confirmed it. Notice how different the framing is for the same underlying issue.

## Interview angle

- "Tell me about a time you disagreed with a senior/decision." → the disagree-and-commit story, with evidence.
- "How do you ask for help?" → the do-the-work-first framing.
- "Tell me about a time you were stuck." → what you tried before asking, and how you framed the ask.
See [`../08-interview-prep/06-behavioral-star-stories.md`](../08-interview-prep/06-behavioral-star-stories.md).
