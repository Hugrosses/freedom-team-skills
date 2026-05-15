---
name: ticket-craft
description: Use this skill every time you create a new ticket, decompose a request into a backlog, or refine an existing ticket. Triggers on any task creation, any "let's add X to the roadmap", any sprint planning. A well-crafted ticket is the difference between an agent shipping in one pass and three rounds of revision. Read this BEFORE writing tickets, not after.
---

# Ticket Craft — How Freedom Tickets Are Written

A ticket is a contract. The owner reads it once, knows exactly what to build, and ships. If a ticket needs Slack threads to be understood, the ticket failed.

---

## The template

Every ticket Aria produces follows this shape:

```
[FRE-NNN] <Imperative title, ≤ 60 chars>

## Story
As a <user role>, I want <capability> so that <outcome>.

## Acceptance criteria
- [ ] <Testable, verifiable, present tense>
- [ ] <Each AC is one fact the implementer must satisfy>
- [ ] <If you can't write a test for it, rephrase it>

## Out of scope
- <Things this ticket does NOT do, to prevent drift>
- <Be specific — "no admin UI for this", "no email notification yet">

## Dependencies
- Blocked by: FRE-NNN <if any>
- Blocks: FRE-NNN <if any>
- Related: FRE-NNN <if any>

## Owner & reviewers
- Owner: <agent>
- Required reviewers: <agents>

## Notes
<Optional. Implementation hints, prior art, gotchas, design decisions already made.>

## Labels
<priority: P0 | P1 | P2>
<size: S | M | L | XL>
<flavor: feature | bug | refactor | infra | spike | docs>
<phase: prototype | beta | launch>
<flags: user-flow? compliance? performance? security?>
```

---

## Field-by-field rules

### Title
- **Imperative.** "Add lead score badge", not "Lead score badge added".
- **Specific.** "Implement Stripe webhook handler" beats "Stripe stuff".
- **≤ 60 characters.** If you can't fit, the ticket is too big — split it.
- **No emojis.** No prefixes like `[FE]` or `[BE]` — labels handle that.

### Story
- One sentence. The classic As/I want/so that structure forces you to say *who* benefits and *why*.
- If you can't articulate a real user benefit, this is probably an infra or refactor ticket — replace the story with a one-line rationale.

### Acceptance criteria
- **Testable.** A reviewer should be able to look at the PR and check each box objectively.
- **Present tense, declarative.** "The badge displays the score as a 2-digit number."
- **Don't describe the implementation.** "Use a div with Tailwind classes" is not an AC. "Score is visible without scrolling on mobile" is.
- **Cover the unhappy path.** "If score is null, the badge shows '—'."
- **Cover empty / loading / error states.** Three lines that catch most regressions.
- **3–8 ACs is the sweet spot.** Fewer and the ticket is under-specified; more and it's two tickets.

### Out of scope
- This is the **anti-creep** field. Use it ruthlessly.
- Examples: "No admin override yet." "No bulk action — single record only." "Email notification is a separate ticket (FRE-145)."
- If you skip this field, the ticket *will* drift. Always include 2–4 lines.

### Dependencies
- Real, not hypothetical. "Blocked by Kai's schema migration" is real. "Blocked by future auth work" is not — that ticket doesn't exist yet.
- Use Multica's link feature so the dependency graph is real.

### Owner & reviewers
- Per the `team-roster` skill. One owner, ≥1 required reviewer for non-trivial work.
- Hugo is a reviewer on: schema/data-loss tickets, anything affecting pricing, anything user-visible at a brand level.

### Labels
- **Priority** — P0 ships this sprint, P1 ships this phase, P2 someday.
- **Size** — S (≤ 4h), M (≤ 1 day), L (≤ 3 days), XL (split it).
- **Flavor** — pick exactly one.
- **Phase** — `prototype`, `beta`, `launch`. Anything past `prototype` doesn't ship yet.
- **Flags** — additive: `user-flow` triggers Ren as reviewer; `compliance` triggers extra audit-trail review; `performance` and `security` trigger extra scrutiny.

### Status on creation
- **New tickets default to `todo`.** Always. A freshly-filed ticket has no work yet — there is nothing to be in-progress on, nothing to review, nothing to block.
- `in_progress` — only after the owner has actually started.
- `in_review` — only after there is something to review (a PR, a spec, a draft). Never on a fresh ticket. This is the most common miscall: a new ticket is *waiting to be picked up*, not *waiting on review*.
- `blocked` — requires a named blocker (another issue, an external dependency). Not a parking spot for "not started yet".
- When in doubt, `todo`. `multica issue create` defaults to `todo` if you omit `--status` — don't override unless one of the above conditions actually holds.

---

## Decomposition rules

When Hugo asks for something larger than a single ticket, decompose. The shape that works for Freedom:

### 1. Identify the surface
Which of the six product surfaces does this affect? (Onboarding, Lead matching, Discovery, Proposal, Client management, Commissions.) If it's none of them, flag scope creep.

### 2. Slice vertically, not horizontally
A vertical slice = one user-perceivable outcome that touches whatever layers it needs. A horizontal slice = "all the backend, then all the frontend" — this is anti-pattern. Always vertical.

✅ "User can see their lead score on the lead detail page" (touches DB + AI + UI in one slice).

❌ "Add lead_score column", "Add scoreLead service", "Add badge component" (three horizontal tickets that ship nothing on their own).

### 3. Each slice ≤ M (≤ 1 day for the owner)
If a slice is L or XL, split again. A series of M tickets is faster than one XL because it ships incremental value and lets reviewers in earlier.

### 4. Sequence on dependencies, not on aesthetics
Sequence the backlog by what blocks what. The first ticket in a chain should unblock the most downstream work.

### 5. Bake in the test ticket
For any vertical slice that touches a critical user flow, add a sibling ticket assigned to Ren: "E2E test for <slice>". Same sprint, blocked-by the slice itself.

### 6. Cap the decomposition
For most asks: 6–12 tickets per decomposition. If you produce >15, you're over-specifying or doing too much at once. Split into phases.

---

## Worked example

**Hugo's ask:** "Add a referral system."

**Bad decomposition** (single ticket):
> Add a referral system. Users get a code, share it, get rewarded.

**Good decomposition** (vertical slices):

```
FRE-201  Define referral schema (codes, redemptions, rewards)    [Kai, M, P1]
FRE-202  Server action: generate referral code on advisor signup [Kai, S, P1]   blocked-by 201
FRE-203  Settings page: show your referral code + share button   [Lior spec → Mira, M, P1]   blocked-by 202
FRE-204  Public referral landing page (claim flow, signup pre-fill) [Lior spec → Mira + Kai, L, P1]   blocked-by 201
FRE-205  Webhook: trigger reward on referee's first paid invoice [Sasha, M, P2]   blocked-by 201
FRE-206  Email: "Your referral just signed up" (Resend template)  [Sasha + Lior, S, P2]   blocked-by 205
FRE-207  Dashboard widget: referral count + pending rewards       [Lior spec → Mira, S, P2]   blocked-by 201
FRE-208  E2E: end-to-end referral signup → reward                 [Ren, M, P1]   blocked-by 204
FRE-209  ADR: referral reward economics (credit vs cash, expiry)  [Aria → Hugo, S, P1]
```

Note:
- ADR ticket exists because reward economics are hard to reverse — Hugo decides.
- Ren has an explicit E2E ticket on the user flow.
- Lior produces specs before Mira implements.
- The dependency graph is explicit.
- Phase 1 = referral works end-to-end. Phase 2 = polish (email, dashboard widget).

---

## Anti-patterns to refuse

You will see these when you decompose. Push back on yourself when you do:

- **Time-boxed tickets** ("Spend 2 days on auth"). Time isn't an outcome. Convert to acceptance criteria.
- **"Investigate X" with no expected output.** A spike is fine, but it ends with a written conclusion — make that the AC.
- **"Refactor for cleanliness"** with no concrete improvement. State the win: "Reduce `apps/web/lib` from 14 files to 6 by consolidating duplicated utilities."
- **"Phase 2" tickets in the active sprint.** They expand and steal capacity. If it's not P0/P1 for this phase, leave it in the backlog.
- **"And also..." tickets.** Two ideas in one ticket. Always two tickets.
- **New ticket filed in `in_review`.** There is nothing to review on a fresh ticket. Default is `todo`. See *Status on creation* above.

---

## When in doubt

Ask: *"Could a specialist agent open this ticket, build it without further questions, and have a reviewer approve it on first pass?"* If no, the ticket needs more work before you assign it.
