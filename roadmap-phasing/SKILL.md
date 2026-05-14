---
name: roadmap-phasing
description: Use this skill whenever you decide whether something belongs in the current phase, when sequencing the backlog, when flagging scope creep, or when responding to "let's also add X". Triggers on every prioritization question, every roadmap conversation, every new ticket that might not belong yet. Read this BEFORE accepting work into the current phase or before pushing back on a request.
---

# Roadmap Phasing — What Ships When

Freedom has three phases. Each has a clear definition of done. Work that doesn't fit the current phase goes to the backlog — not into the active sprint.

The phases exist because Hugo has ~3–4 hours per evening, and the team has finite focus. Saying "yes" to everything in phase 1 is how prototypes never ship.

---

## Phase 1 — **Prototype** *(current phase)*

**Goal:** A clickable, end-to-end Freedom Portal that demonstrates the full advisor lifecycle on real data, deployed to a staging URL Hugo can show to potential users. Not yet billable. Not yet certified.

**Definition of done:**
- An advisor can sign up, complete KYC (mocked verification), and reach the dashboard.
- An advisor can receive at least one matched lead (seeded), open it, run a discovery meeting (with notes), and generate a proposal.
- A proposal can be e-signed via DocuSeal in a sandbox flow.
- A signed proposal creates a client record with a portfolio snapshot.
- The commission dashboard renders mock retrocession data.
- All six lifecycle surfaces have at least the "happy path" working.
- The visual system matches the existing mockup's aesthetic (navy + gold + Cormorant) and feels distinct from Salesforce/shadcn defaults.
- Supabase RLS is on for every table. EU residency. Audit log records key actions.
- Playwright E2E covers the happy paths. Vitest covers business logic.
- Stripe is integrated in **test mode** with all three plans, but no real billing.

**Phase 1 explicitly excludes:**
- Real KYC provider integration. Mocked.
- Real custodian integration (Swissquote, Saxo). Mocked.
- Real lead-generation marketplace. Seeded.
- Live video infrastructure at scale. LiveKit/Daily integrated, not load-tested.
- AI cost optimization. Default to Sonnet 4.6 everywhere; tune later.
- Multi-language beyond EN/FR. No IT/DE.
- Mobile native apps. Web only, mobile-responsive.
- Admin / back-office tooling for Hugo himself. CLI scripts are fine.
- Compliance certifications. Architected for, not certified.
- Performance optimization beyond "feels snappy on a M1 MacBook on fast wifi".
- White-glove onboarding flows, custom branding for Elite tier. Single template.

**Estimated horizon:** 8–12 weeks of agent-team output, depending on Hugo's review velocity.

---

## Phase 2 — **Beta**

**Goal:** A version Hugo can put in front of 5–10 real IFAs in Romandie, with real money flowing for subscription only (not yet for managed assets).

**Triggers entering phase 2:**
- Phase 1 is done. All ACs above checked.
- Hugo has demoed the prototype to ≥3 real advisors and gotten qualitative validation.
- Hugo has identified the top 3 friction points to fix before letting beta users in.

**What gets built in phase 2:**
- Real KYC provider (Sumsub or Onfido — Sasha + Aria choose).
- Real custodian read-only integration (Swissquote API for portfolio data — start with one).
- Stripe live mode. Real subscriptions. EU VAT.
- Email deliverability hardened (DKIM, SPF, DMARC, dedicated Resend domain).
- Onboarding polished based on real-user friction logs.
- Lead marketplace MVP (real prospects, even if seeded by Hugo's network).
- Compliance review with a Swiss legal advisor (paid engagement).
- SLA target: 99% uptime, <500ms p95 page loads.
- Sentry alerting wired to Hugo's phone for P0 errors.
- Status page (Statuspage or similar).

**Phase 2 still excludes:**
- SOC 2 / ISO 27001.
- Multi-region beyond EU-Frankfurt.
- IT/DE localization.
- Native mobile apps.
- Open API for third-party integrations.
- Marketplace for leads beyond Hugo's curation.

**Estimated horizon:** 8–12 weeks after phase 1.

---

## Phase 3 — **Launch**

**Goal:** Public availability. Self-serve signup. Marketing site. Real growth motion.

**What's in phase 3:**
- Public marketing site (separate from the app).
- Self-serve signup with no Hugo gating.
- Italian and German localization.
- IT/DE expansion: Ticino, then Geneva-region cross-border, then full DACH.
- Native mobile apps (iOS first).
- Open API for selected partners.
- Compliance certs as needed by jurisdiction.
- Affiliate / referral program live.
- Content marketing flywheel.

**Phase 3 is a planning placeholder.** Don't over-design for it now. Re-plan when phase 2 closes.

---

## How to use this skill

### When a ticket comes in
1. Read the ticket. Read the surface and acceptance criteria.
2. Ask: *Is this in phase 1's "definition of done"? Or is this in "phase 1 excludes"?*
3. If unclear: *Does this materially advance the prototype demo?*
4. If still unclear: ask Hugo, batched with other open questions.

### When Hugo asks for something out-of-phase
- **Don't reflexively say no.** Hugo has product instincts; there might be a reason.
- **Ask why.** "This is currently scoped to phase 2 — what's making it urgent?"
- **Offer alternatives.** "Could we ship a stub for phase 1 and the real thing in phase 2?"
- **If yes, write an ADR.** Pulling phase 2 work into phase 1 changes the timeline. Document the trade-off.

### When you spot scope creep
- Flag it explicitly. "FRE-NNN as written includes phase 2 work — specifically [X]. Scope to just [Y] for now?"
- Don't just silently expand the ticket. The ticket is the contract; if it changes, the contract changes.

### When the team is ahead of schedule
Lucky problem. Pull the next P1 from the backlog — not from phase 2. Phase order matters because earlier surfaces inform later ones; jumping ahead causes rework.

---

## What "shipped" means at each phase

- **Prototype:** deployed to `staging.freedom.app` (or equivalent), accessible via password, doesn't break Playwright suite.
- **Beta:** deployed to `app.freedom.app`, real users have access, errors monitored, weekly retrospective with Hugo.
- **Launch:** publicly accessible, paid, supported. Different operational standard. Don't conflate with prototype.

The single most common mistake is treating the prototype like the launch. Don't. The prototype's job is to be *demoable*, not *defensible*.
