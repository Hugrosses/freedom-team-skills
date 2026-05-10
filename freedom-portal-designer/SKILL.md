---
name: freedom-portal-designer
description: Use this skill when navigating the existing 9 311-line mockup, when picking which mockup section to reference in a spec, and when discussing aesthetic decisions Hugo has already made. Triggers on every reference to docs/reference/freedom-portal.html and every "what was Hugo's vision" question.
---

# freedom-portal-designer — The Mockup as Source

> **Note for Aria + Hugo:** the `team-roster` skill marked this as "(existing)" but no canonical content was found anywhere — no local file under `~/.claude/skills/`, no entry in this repo prior to this PR, no upstream link. Treating it as a *new* skill that Lior owns going forward.
>
> If a previous version exists somewhere you know about, point me at it and I'll merge / supersede in a follow-up PR. Otherwise this stub is the seed.

## What this skill is for

The existing mockup at `docs/reference/freedom-portal.html` is the closest written record of Hugo's aesthetic and product voice. 9 311 lines, single file, vanilla JS + Tailwind CDN. It contains:

- Every visible surface in the prototype's intended scope
- Hugo's calibration of the navy/gold/Cormorant palette in real composition
- The microcopy Hugo himself wrote (chips, headers, testimonials, error states)
- Interaction patterns (the in-meeting risk quiz, the kanban drag-drop, the proposal builder)
- Edge cases he cared enough to mock (white-label preview, EN/FR toggle, pitch mode)

This skill is the **map** to the mockup — where to look for what.

## The non-negotiables

- **The mockup is canonical until Lior produces a successor spec for that surface.** When the mockup conflicts with a Lior spec, the spec wins (it's the more recent decision). When the mockup conflicts with anything *else* (a Mira instinct, an Aria suggestion), the mockup wins until Lior says otherwise.
- **Reference by line number** in specs and in PR discussions. `freedom-portal.html` lines NN–MM is unambiguous; "the lead detail page" is not.
- **Hugo's microcopy is canonical** for the surfaces it covers. When Lior writes a fresh spec for a surface the mockup already has copy for, default to the mockup's words and only deviate with an explicit "I'd change this to X because Y" line.
- **Don't reproduce mockup bugs.** The mockup is a vanilla-JS simulator with seed data; some interactions misbehave. When Mira finds a mismatch between the mockup and what RSC + Supabase can deliver, that's a clarification request to Lior, not a faithful port.

## Map of the mockup (lines, approximate)

> **TODO (Lior, your audition or shortly after):** confirm or correct the line ranges below with a fresh read. They were derived during Aria's IAF-1 audit and may have drifted as Hugo iterated.

```
Lines    1– 200  : <head>, CSS variables, Tailwind config, font loading
Lines  214– 545  : state object, seed data (leads, products, templates, etc.)
Lines  594– 800  : Landing page (hero, "how it works", feature split, pricing, testimonials)
Lines 1052–1247  : Onboarding (Welcome, KYC, Profile, Hours, Compliance review)
Lines 1247–1373  : AppShell (sidebar, mobile bottom nav, topbar)
Lines 1373–1565  : Home dashboard (KPIs, week agenda, mini-leads, first-client card)
Lines 1566–1710  : Leads list / detail / accept-lead flow
Lines 1716–2440  : Meetings (schedule, room, screen-share, in-meeting quiz, AI summary)
Lines 2451–2772  : Clients list / detail / Proposals list
Lines 2779–3170  : Proposal builder (plan picker, mix, projection, preview, send, sign)
Lines 3240–3697  : Commissions, Analytics
Lines 3697–3995  : Academy, Settings
Lines 4117–4429  : Leads Kanban + Marketplace public + advisor public page
Lines 4429–5341  : Client portal (home, portfolio, documents, messages, referrals)
Lines 5341–5677  : Team view (Elite multi-seat) + Referrals
Lines 5585–6071  : Audit log + Billing (invoices, plan switcher, payment method)
Lines 6071–6671  : Plan comparator + command palette + notifications + new menu + product tour
Lines 6698–6948  : Risk quiz, AI transcription, AI proposal, follow-up writer
Lines 6956–7444  : Persona widget, products catalog, product detail, mix picker
Lines 7470–7760  : Templates library, Integrations, Planning simulator
Lines 7980–8590  : Swiss insurance hub (LAMal sim, LCA info, 3a sim, 3b info)
Lines 8643–9300  : White-label preview, pitch mode, language application
```

## How to use the mockup in a spec

1. **Identify the surface** in the ticket. Find the matching range above (or correct it if drifted).
2. **Read the relevant lines** — both the rendering (the JS function and the HTML it produces) and the copy/microcopy.
3. **Note what's portable** vs what's mockup-specific (e.g., the vanilla-JS state machine is mockup-specific; the visual composition isn't).
4. **Cite line numbers** in your spec so Mira can sanity-check.
5. **Annotate deviations** when you choose to depart from the mockup. "Mockup line 1399 shows a gold-bordered KPI card; I'm dropping the gold border in favor of `--line` because the card sits inside a gold-CTA hero and the contrast collapses. Mira: use `--line`, not `--gold`."

## Patterns to author here later

- The "Hugo's voice" copy patterns — micro-headlines, chip text, testimonial cadence, error-state warmth.
- The motion patterns Hugo iterated on (the compliance-review animation, the confetti, the persona-widget pulse).
- The component archetypes (KPI card, MiniLead, FirstClientCard, plan card, etc.) and their token usage.
- The "this surface hasn't been mocked" list — what Lior needs to design from scratch (anything in phase 2/3).

Author in full once Lior has navigated the mockup for 3+ specs and we know which sections of this map were useful vs theater.
