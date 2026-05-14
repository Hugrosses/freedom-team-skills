---
name: tech-stack-canon
description: Use this skill whenever a technical choice comes up — framework, library, tool, folder structure, naming convention, code style, package manager, deployment target. Triggers on any "should we use X" question, any new dependency proposal, any boilerplate or scaffolding ticket. Read this BEFORE writing any ADR about technology, and BEFORE assigning a ticket that introduces a new tool. Re-litigating locked decisions wastes everyone's cycles.
---

# Tech Stack Canon — Locked Decisions for Freedom Portal

This is the locked stack for the Freedom prototype. Every entry below has been chosen deliberately. **Do not re-open these decisions** unless something has materially changed. If you genuinely need to override one, write an ADR explaining why.

---

## The stack at a glance

| Layer | Choice | Status |
|---|---|---|
| **Frontend framework** | Next.js 15 (App Router, RSC) | Locked |
| **Language** | TypeScript, strict mode | Locked |
| **Styling** | Tailwind CSS 4 | Locked |
| **Component primitives** | shadcn/ui — heavily customized | Locked |
| **Animation** | Framer Motion | Locked |
| **Icons** | Lucide | Locked |
| **Backend / DB** | Supabase (Postgres + Auth + Storage + Edge Functions + Realtime) | Locked |
| **AI / LLM** | Anthropic API (Claude Sonnet 4.6 default; Haiku for cheap tasks; Opus for hard ones) | Locked |
| **Billing** | Stripe (subscriptions, EU VAT) | Locked |
| **Email** | Resend | Locked |
| **Video meetings** | LiveKit *or* Daily — Sasha owns choice, Aria reviews | Open ADR |
| **E-signature** | DocuSeal (self-hosted, EU region) | Locked |
| **Calendar embed** | Cal.com | Locked |
| **Hosting (web)** | Vercel | Locked |
| **Hosting (DB/backend)** | Supabase Cloud, EU-Frankfurt region | Locked |
| **Package manager** | pnpm | Locked |
| **Monorepo tool** | Turborepo | Locked |
| **Linter / formatter** | Biome | Locked |
| **Unit / integration tests** | Vitest | Locked |
| **E2E tests** | Playwright | Locked |
| **CI** | GitHub Actions | Locked |
| **Observability** | Sentry (errors), Vercel Analytics (web vitals) | Locked |

---

## Folder structure (monorepo)

```
freedom/
├── apps/
│   └── web/                 # Next.js app (Mira's primary domain)
│       ├── app/             # App Router routes
│       ├── components/      # app-specific components
│       └── lib/             # app-specific utilities
├── packages/
│   ├── ui/                  # shared design system (Lior + Mira)
│   │   ├── components/      # primitive + composite components
│   │   ├── tokens/          # design tokens (colors, typography, motion)
│   │   └── styles/          # global CSS, Tailwind preset
│   ├── db/                  # Supabase types + clients (Kai)
│   │   ├── client.ts
│   │   ├── types.ts         # generated from Supabase
│   │   └── queries/         # typed query helpers
│   ├── ai/                  # Anthropic services & prompts (Vera)
│   │   ├── prompts/         # versioned prompt files
│   │   ├── services/        # lead-scoring, transcription, etc.
│   │   └── evals/           # eval datasets + runners
│   ├── integrations/        # Stripe, Resend, video, e-sign (Sasha)
│   └── lib/                 # cross-cutting utilities, types
├── supabase/                # Kai's primary domain
│   ├── migrations/
│   ├── functions/           # edge functions
│   └── seed.sql
├── e2e/                     # Playwright tests (Ren)
└── docs/
    ├── adr/                 # Architectural Decision Records
    └── README.md
```

Ownership maps to agents. Cross-package work needs Aria's routing.

## Naming conventions

- **Files:** `kebab-case.ts` for modules, `PascalCase.tsx` for React components.
- **Components:** `PascalCase`, descriptive — `LeadScoreCard`, not `Card1`.
- **Hooks:** `useThing.ts`, lowercase verbs — `useAdvisor`, `useLeads`.
- **Server actions:** verb-first — `createProposal`, `signProposal`, `archiveLead`.
- **DB tables:** `snake_case`, plural — `advisors`, `leads`, `proposals`, `meeting_notes`.
- **DB columns:** `snake_case` — `created_at`, `aum_chf`, `is_signed`.
- **Env vars:** `SCREAMING_SNAKE_CASE`, prefixed by service — `SUPABASE_URL`, `STRIPE_SECRET_KEY`, `ANTHROPIC_API_KEY`.
- **Branches:** `<agent>/<ticket-id>-<short-slug>` — e.g. `mira/FRE-142-hero-section`.
- **Commits:** Conventional Commits — `feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, `test:`.

## Code style — locked rules

1. **TypeScript strict mode is on.** No `any`. If you must, use `unknown` and narrow.
2. **Server Components by default.** Add `"use client"` only when you genuinely need it (state, effects, browser APIs, event handlers on intrinsic elements that need React).
3. **Server Actions for mutations.** API routes only when an external service needs to call us (webhooks).
4. **No client-side data fetching libraries** (no SWR, no React Query) for the prototype — RSC + Server Actions cover it. Revisit if we add real-time complexity beyond what Supabase Realtime handles.
5. **Tailwind utility classes inline.** No `@apply` except in shared component CSS. No CSS-in-JS.
6. **Design tokens as CSS variables** (matching the existing mockup's pattern). Tailwind reads from them via the preset in `packages/ui/styles`.
7. **Zod for runtime validation** at every trust boundary (form input, API response, webhook payload).
8. **No `console.log` in committed code.** Use the structured logger in `packages/lib/logger`.

## Database conventions

1. **RLS on every table.** No exceptions. Kai signs off.
2. **Audit log table** (`audit_events`) records every mutation that affects compliance-relevant data (advisor profile, KYC, proposals, signatures, commissions). Schema includes actor, action, target, timestamp, IP, user agent.
3. **Soft delete** (`deleted_at` timestamp) on user-facing records — clients, leads, proposals. Hard delete only via explicit GDPR erasure flow.
4. **Timestamps:** every table has `created_at` and `updated_at`, both `timestamptz`, defaulted server-side.
5. **IDs:** UUID v7 (sortable). Never expose internal sequential IDs.
6. **Migrations are forever.** Test on a copy before applying. No data loss without explicit Hugo approval.

## AI / LLM conventions

1. **Default model:** Claude Sonnet 4.6 (`claude-sonnet-4-6`). Haiku for cheap, latency-sensitive tasks (classification, lead scoring at scale). Opus only when explicitly justified.
2. **Prompts are code.** Version them in `packages/ai/prompts/`. Diff every change.
3. **Every AI feature has an eval.** Vera owns this. No prompt ships without a baseline eval.
4. **Streaming by default** for any AI feature with user-perceived latency >1s.
5. **Cost ceiling per feature** declared at design time. If a feature exceeds ceiling, escalate to Aria.
6. **No PII in prompts** beyond what's needed for the task. Strip before sending.

## Security & compliance defaults

1. **All routes are auth-gated** unless explicitly marked public (landing, marketing, signup).
2. **CSRF protection** via Server Actions (Next handles it). Webhooks verify signatures.
3. **Secrets in environment only.** Never committed. Vercel/Supabase env management.
4. **No client-side secrets.** API keys for third parties live in Server Actions or edge functions.
5. **Rate limit auth endpoints** and AI-cost-bearing endpoints. Use Upstash if needed.
6. **EU data residency only** for the prototype. Reject any tool that doesn't offer it.

## When to write an ADR vs just decide

Write an ADR if:
- The decision changes the table above.
- The decision affects more than one agent's domain for >1 sprint.
- The decision is hard to reverse (DB migration with data, billing logic, auth model).

Don't write an ADR for:
- Choosing between two equivalent libraries within an unimportant domain. Pick one, move on.
- Naming a single component.
- Picking a Tailwind value.

When in doubt, default to *smaller* documents — a one-paragraph note in the ticket beats a four-page ADR.
