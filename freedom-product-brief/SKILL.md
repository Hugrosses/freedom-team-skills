---
name: freedom-product-brief
description: Use this skill whenever you need to ground a decision, ticket, or response in what Freedom Portal actually is. Triggers on any question about the product's purpose, target user, scope, positioning, pricing, feature priority, or "is this in scope". Read this BEFORE any decomposition, ADR, or strategy reply. If you find yourself proposing a feature without checking this skill, stop and re-read it.
---

# Freedom Portal — Canonical Product Brief

This is the source of truth for *what Freedom is*. If a fact about the product isn't in this skill or hasn't been confirmed by Hugo, treat it as an **open question** — do not invent.

---

## One-line positioning

**Freedom is the operating system for independent financial advisors managing CHF 500k to a few million in AUM — too big for spreadsheets, priced out of Avaloq.**

## The wedge

Independent Financial Advisors (IFAs) in this segment have been forced to choose between:
- **Consumer tools** (Notion, Google Workspace, generic CRMs) that aren't compliant and don't speak their craft.
- **Enterprise tools** (Salesforce Financial Services Cloud, Avaloq, Wealth Dynamix) that cost five figures per month + integrators and assume a bank-sized practice.

Freedom collapses that gap. Compliant by default, priced for solo-to-small-team practices, designed for advisors not for IT departments.

## Target user

The **Independent Financial Advisor** running:
- Solo or with 1–5 colleagues.
- CHF 500k to a few million CHF in AUM (low end at signup, growing).
- 10–80 active clients.
- Based in Switzerland (Romandie first), then France, then broader EU.
- French native or French + English working language.
- Cares about: client relationships, calm professionalism, brand. Does *not* care about: dashboards full of metrics, AI gimmicks, "growth hacks".

The reference avatar in our existing mockup is **Claire Fontaine** — 37 clients, €8.4M AUM, 96% retention. Build for her.

## Pricing (locked for prototype)

| Plan | Price | Tag |
|---|---|---|
| **Starter** | CHF 149/mo | Solo advisor, getting started. Up to 15 active clients. |
| **Standard** | CHF 299/mo | The growing practice. Unlimited clients. *Featured tier.* |
| **Elite** | CHF 449/mo | Teams, boutiques, concierge. 5 seats included. |

All plans: 14-day free trial, MiFID II compliance tooling, bank-grade encryption, EU data residency.

These prices are **set for the prototype**. Revisiting requires Hugo's explicit sign-off.

## Core feature spine (in delivery order)

The advisor's lifecycle is the spine. Build in this order:

1. **Onboarding** — KYC, license verification, compliance review, practice profile, calendar.
2. **Lead matching** — AI-scored prospects matched against the advisor's specialties. Quality over quantity.
3. **Discovery** — Calendar, secure video, shared notes, risk tolerance quiz.
4. **Proposal** — AI-generated from the discovery, white-labeled, e-sign on any device.
5. **Client management** — Portfolio snapshot, timeline, secure messaging, document vault.
6. **Commissions & growth** — Commission dashboard, retention signals, referral engine, marketplace listing.

If a ticket doesn't fit one of these six surfaces, it's probably out of scope for the prototype.

## Aesthetic & brand (non-negotiable)

This is the moat. Treat as immutable unless Hugo or Lior changes it:
- **Palette:** Navy 900 (`#070e1d`) → Navy 500 (`#1e3356`) base; Gold (`#d4af37`) accents; Cream (`#f5f1e8`) text.
- **Typography:** Cormorant Garamond for serif headings, Inter for body and UI.
- **Tone:** "Calm UI, zero noise." Notifications are earned, not pushed. Compliance is invisible.
- **Reference:** the existing 9000-line HTML mockup (`freedom-portal.html`). When in doubt, that file is the visual ground truth until Lior produces successor specs.
- **Forbidden:** Salesforce-look (dense data tables, blue gradients, dashboard maximalism), generic shadcn (default Inter-only, neutral grays, no character), trendy AI-product look (purple gradients, glow buttons everywhere).

## Compliance surface (prototype scope)

The prototype must architect for compliance from day one, even though formal certification is deferred:
- **MiFID II / AMF / FINMA** — relevant suitability, KYC, and record-keeping concepts must be reflected in the data model.
- **GDPR** — EU-only data residency, right-to-erasure-friendly schema, audit log on every meaningful action.
- **Storage** — Sensitive documents encrypted at rest. No PII in logs.

What we are *not* doing in the prototype: SOC 2 audit, penetration tests, formal compliance review with a regulator. Those are post-MVP.

## Languages

- **English** is the default for the codebase, tickets, ADRs, and team communication.
- **French (Switzerland/France)** is the default for the *user-facing product*. The mockup already has an EN/FR toggle — keep that pattern.
- Italian and German are on the roadmap; not for the prototype.

## What Freedom is NOT

Saying no is part of the spec. The prototype is **not**:
- A trading platform. No order execution, no live market data feeds.
- A custodian. We don't hold assets; we sit alongside the advisor's existing custody (Swissquote, Saxo, etc.).
- A robo-advisor. The advisor is the human in the loop; we make their work better, not replace them.
- A general CRM. We are deeply opinionated about *advisor* workflow.
- A marketing site builder. The "marketplace listing" is a directory, not a Squarespace.

If a ticket drifts toward any of the above, flag it as **scope creep** and bring it to Hugo.

## Reference artifacts

- `freedom-portal.html` — the 9000-line single-file mockup. Visual + interaction ground truth.
- `freedom-portal-designer` skill (Lior's) — the design-system source of truth.
- This skill — the product source of truth.

When these conflict, escalate to Hugo. Do not silently reconcile.
