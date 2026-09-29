# Code Review Mindset

Reviewing well is a mid-level skill and an interview topic ("here's a PR, review it"). The goal isn't to find fault — it's to protect the codebase while respecting the author.

## The five layers of review (in order)

Review in this order; stop escalating severity once a layer passes.

1. **Does it work?** Does it do what the PR says? Run it / read the logic.
2. **Is it correct?** Edge cases, error paths, the failure modes. (This is where bugs hide.)
3. **Will it stay correct?** Is it tested? Does it encode an invariant future changes might break?
4. **Does it fit?** Does it follow the repo's patterns, or invent a new way to do a solved problem?
5. **Is it kind to future maintainers?** Naming, clarity, a comment where the *why* isn't obvious.

A junior reviewer jumps to layer 5 (style nits). A senior reviewer weights layers 2–3 and treats 5 as optional polish.

## Severity grading (say which you mean)

- **Blocking:** correctness or security defect; must fix before merge.
- **Important:** maintainability/risk; should fix, can discuss.
- **Optional / nit:** preference; author's call.

Always label. "This is a nit, feel free to ignore" and "Blocking: this leaks another tenant's data" are very different, and the author needs to know which you're saying.

## A repo-specific review checklist for Umami

When reviewing a change here, ask:

- [ ] **Authorization:** does every id-bearing route call a `can*` check against the URL id? (The #1 risk here — [security checklist](../05-quality-engineering/05-security-checklist.md).)
- [ ] **Both SQL branches:** if a query changed, did *both* the Prisma and ClickHouse implementations change together? (The #1 correctness trap.)
- [ ] **Validation:** does new input pass a Zod schema? Did anything bypass it (suffixed filters, `anyObjectParam`)?
- [ ] **Public contract:** does this change the `/api/send` payload shape or a column width that deployed trackers depend on? If so, is it backward compatible?
- [ ] **SQL safety:** any user value in a SQL template string (vs. bound param)? Any dynamic identifier not allow-listed?
- [ ] **Error handling:** are errors returned to clients generic (not leaking `serializeError` detail)?
- [ ] **Side effects:** any critical write in an empty catch that returns success?
- [ ] **Tests:** is the deny/failure path tested, not just the happy path?
- [ ] **Cache:** if a value that's cached changed, is the cache invalidated on write?
- [ ] **`||` vs `??`:** any place a valid `0`/`''` could be wrongly defaulted?

## Example comments (kind + specific)

Bad (vague, unkind): *"This is wrong."*
Better: *"Blocking: this returns before the `canViewWebsite` check, so any authed user can read any website's data. Move the check above the query like in `realtime/[websiteId]/route.ts:20`?"*

Bad (nit as blocker): *"Rename this variable."*
Better: *"Optional nit: `d` is a little terse here — maybe `dateRange`? Not blocking."*

Good (teaching, not just flagging): *"Important: `orderBy` gets interpolated into the SQL string in `pagedRawQuery` (prisma.ts:318), so passing user input here is an injection risk. Could we allow-list the sortable columns? Happy to pair."*

Good (approving well): *"LGTM — clean extraction, follows the existing constants pattern, and the fallback is preserved. Ship it."*

## Principles

- **Review the code, not the person.** "This function…" not "You always…"
- **Ask, don't command,** when it's a judgment call: "What do you think about…?" invites a conversation; "Change this" shuts it down.
- **Explain the why,** with an anchor. A review comment that teaches is worth ten that just flag.
- **Approve when it's good.** A reviewer who never approves trains authors to route around them.
- **Pick your battles.** Don't bury a blocking security issue under twenty style nits.

## The reviewer's version of "measure first"

Before writing a comment, confirm you're right: re-read the code, check the anchor, consider the case where you're wrong. A confidently-wrong review comment costs the author time and costs you credibility. "I might be missing something, but doesn't this…?" is both humbler and safer than a wrong assertion.

## Drill

Take any kata from [`../04-code-reading-gym/04-review-katas.md`](../04-code-reading-gym/04-review-katas.md), write your three findings, label each severity, and rewrite one as a kind, specific, anchored comment. Compare against the expected findings. Then do the hardest thing: on Kata 7 (the good PR), practice writing an *approving* comment that still adds value.

## Interview angle

- "Here's a PR — review it." → walk the five layers aloud, label severity, cite anchors, be kind. See [`../08-interview-prep/05-debugging-and-code-review-rounds.md`](../08-interview-prep/05-debugging-and-code-review-rounds.md).
- "Tell me about a time you gave difficult feedback." → a STAR story from a real review.
- "How do you review a large PR?" → the layered order, and asking for it to be split if it's too big to review well.
