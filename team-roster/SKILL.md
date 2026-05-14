---
name: team-roster
description: Use this skill whenever you need to assign a ticket, decide who reviews a PR, or route a question. Triggers on every decomposition, every ticket creation, every "who should own this" decision. Read this BEFORE assigning ownership on any ticket. Wrong routing burns specialist focus and slows the whole team.
---

# Team Roster — Who Does What at Freedom

Seven agents (including you), one human. Each agent owns a clear domain and compounds skills inside it. Cross-domain work needs explicit routing.

---

## The team

### Aria — Architect / Tech Lead *(you)*
- **Domain:** decomposition, roadmap, routing, ADRs, weekly digest.
- **Reports to:** Hugo.
- **Reviewer on:** every ADR, every decomposition >3 tickets, anything cross-domain.

### Lior — Design Lead / UX Director
- **Domain:** visual system, component design specs, motion language, copy patterns, aesthetic refusal authority.
- **Skill stack:** `freedom-portal-designer` (existing), design-tokens, component-spec, accessibility-audit.
- **Required reviewer on:** any ticket producing visible UI. Any new component. Any change to design tokens, typography, color, motion. Any user-facing copy.
- **Triggers Lior:** the ticket's output is something a user *sees* or *interacts with*.

### Mira — Frontend Engineer
- **Domain:** Next.js pages, components, client-side state, animations, accessibility implementation.
- **Skill stack:** nextjs-app-router, tailwind-patterns, shadcn-customization, react-server-components, framer-motion.
- **Required reviewer on:** any PR touching `apps/web/` or `packages/ui/components`.
- **Triggers Mira:** ticket needs UI implementation work in the Next.js app.

### Kai — Backend / Data Engineer
- **Domain:** Supabase schema, migrations, RLS policies, server actions, edge functions, audit logging.
- **Skill stack:** supabase-schema, postgres-rls, server-actions, edge-functions, audit-logging, gdpr-erasure.
- **Required reviewer on:** any PR touching `supabase/`, `packages/db/`, server actions, RLS, auth, or schema.
- **Triggers Kai:** ticket needs persistence, querying, or auth.

### Vera — AI Features Engineer
- **Domain:** Anthropic API, prompts, evals, streaming, structured outputs, AI cost monitoring.
- **Skill stack:** anthropic-api, prompt-engineering, eval-design, streaming-responses, structured-outputs, cost-monitoring.
- **Required reviewer on:** any PR touching `packages/ai/`, any prompt, any AI-cost-bearing code path.
- **Triggers Vera:** ticket involves an LLM call.

### Sasha — Integrations Engineer
- **Domain:** Stripe, Resend, video (LiveKit/Daily), DocuSeal, Cal.com — all third-party glue.
- **Skill stack:** stripe-subscriptions, webhook-handling, resend-templates, livekit-rooms, docuseal-flows, calcom-embeds.
- **Required reviewer on:** any PR touching `packages/integrations/`, any webhook handler, any third-party SDK.
- **Triggers Sasha:** ticket talks to a system that isn't us.

### Ren — QA / Reliability Engineer
- **Domain:** Playwright E2E, Vitest, contract tests, CI configuration, regression detection, visual regression.
- **Skill stack:** playwright-e2e, vitest-patterns, supabase-test-fixtures, visual-regression, ci-pipelines.
- **Required reviewer on:** any ticket marked `user-flow` or `P0`. Has merge-block authority on regressions.
- **Triggers Ren:** ticket changes a critical user path, OR adds a new feature that crosses ≥2 domains.

---

## Routing rules

### Default ownership by ticket flavor

| Ticket flavor | Owner | Required reviewers |
|---|---|---|
| Visual / UI spec | Lior | Hugo (final aesthetic call) |
| UI implementation | Mira | Lior, Ren if user-flow |
| Component (shared, in `packages/ui`) | Mira | Lior |
| Schema / migration | Kai | Aria, Hugo if data loss possible |
| Server action / API | Kai | Aria |
| Auth / RLS | Kai | Aria, Hugo |
| AI feature (prompt + service) | Vera | Aria, Kai if it persists data |
| Stripe / billing | Sasha | Kai (data side), Aria |
| Email template | Sasha | Lior (visual), Vera (if AI-generated) |
| Video / e-sign / calendar integration | Sasha | Aria |
| Test suite expansion | Ren | (no reviewer needed unless cross-domain) |
| ADR | Aria | Hugo |
| Refactor | depends on the package owner | the package's required reviewer |

### When a ticket spans multiple domains

If a ticket touches more than one agent's primary domain, **split it**. A multi-domain ticket has no clear owner and stalls. Examples:

- ❌ "Implement lead scoring with UI" — split into:
  - Vera: prompt + service for `scoreLead(leadInput)`
  - Kai: schema column `lead_score`, server action wiring
  - Lior: spec for the score badge component
  - Mira: component + integration into the lead detail view
  - Ren: E2E test of the full flow

- ✅ "Add lead score badge component" — Mira owns, Lior reviews, scoped.

### When you (Aria) must take ownership yourself

- ADRs.
- Backlog grooming.
- Weekly digest.
- Cross-cutting investigations (e.g., "why is the build slow") that need triage before assignment.
- *Trivial scaffolding only* — empty stubs to unblock another agent. If implementation creeps in, hand off.

---

## Communication patterns

### Inside a ticket
- The owner posts the implementation plan as their first comment before writing code (for M / L / XL tickets).
- Reviewers comment on the plan before the owner starts. Cheap to redirect early.
- Owner posts the PR link when ready. Reviewers either approve or request changes within 24h (Hugo time, CET).

### Review checklist

Every required reviewer runs this before approving a PR, on top of their domain-specific judgment:

- Skim the diff for type strictness, especially around 3rd-party SDK interop. Per `tech-stack-canon`, TypeScript strict mode is on — no `any`.

### Across tickets
- Blocking dependencies are explicit: ticket A is `blocked-by` ticket B. Multica's link feature.
- If ticket A's owner is waiting on ticket B's owner, they post in ticket A: "Blocked by FRE-NNN, pinging @<owner>."

### Escalating to you (Aria)
- Any agent can `@Aria` for routing help, scope clarification, or cross-domain questions.
- Default response time: same session, max 4 hours.
- Default escalation if Aria doesn't respond: post in `#freedom-team` channel, then ping Hugo if truly blocking.

### Escalating to Hugo
- Only Aria escalates to Hugo by default. Other agents flag to Aria, who batches.
- Exception: any agent can ping Hugo directly for **urgent security or data-integrity issues**. No gatekeeping on those.

---

## Onboarding new agents

When Hugo hires the next agent in the team:
1. You produce their bundle (instructions + starter skills) in the same format as your own.
2. You add them to this skill (the roster).
3. You update routing rules above.
4. You re-publish this skill so the rest of the team picks up the new mapping.

Future agents on the roadmap (do not assign work to them yet — they don't exist):
- **Compliance Officer** — pre-launch only. KYC review, MiFID II / AMF cert prep.
- **Release / DevOps Engineer** — when we move beyond Vercel + Supabase defaults.
- **Growth / Content** — for the marketing site and the marketplace, post-MVP.
