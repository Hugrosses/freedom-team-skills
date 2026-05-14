---
name: nextjs-app-router
description: Use this skill before adding a route, before deciding Server vs Client Component, before introducing a state library, and before changing the apps/web folder layout. Triggers on every new page, every "use client" decision, every layout / loading / error boundary, every Next.js dependency change.
---

# Next.js App Router — Freedom's UI Conventions

This is the source of truth for *how* we write Next.js 15 at Freedom. The conventions are deliberately narrow: fewer choices means fewer style debates, faster reviews, and a codebase Mira can grow without re-litigating fundamentals on every PR.

If you find yourself wanting to break a rule below, write a one-paragraph note in the PR description explaining why. If the explanation generalizes, propose updating this skill.

---

## The non-negotiables

1. **Server Components by default.** `"use client"` only when you genuinely need state, effects, browser APIs, or event handlers on intrinsic elements that need React (form `onSubmit`, button `onClick`).
2. **Server Actions for mutations.** API routes are only for inbound webhooks the outside world calls.
3. **No client-side data fetching libraries.** RSC + Server Actions cover phase 1. No SWR, no React Query, no tRPC.
4. **Tailwind utility classes inline.** No CSS Modules, no styled-components, no `@apply` outside `packages/ui/styles`.
5. **All committed copy strings live in `messages/` translation files.** Hard-coded strings get caught by lint.
6. **No `console.log` in committed code.** Use `packages/lib/logger`.

---

## Folder layout

Inside `apps/web/`:

```
apps/web/
├── app/
│   ├── [locale]/                # Path-based locale segment — fr-CH default, en alongside
│   │   ├── (public)/            # Routes that don't require auth
│   │   │   ├── page.tsx         # Landing
│   │   │   └── _components/
│   │   ├── (advisor)/           # Routes for authenticated advisors
│   │   │   ├── layout.tsx       # AppShell wrapping
│   │   │   ├── home/page.tsx
│   │   │   ├── leads/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── proposals/page.tsx
│   │   │   ├── clients/page.tsx
│   │   │   └── _components/
│   │   ├── (client)/            # Routes for client portal (P1)
│   │   │   ├── layout.tsx
│   │   │   └── ...
│   │   ├── layout.tsx           # Locale layout — sets <html lang>, loads messages
│   │   └── not-found.tsx
│   ├── api/                     # Webhooks only — never user-handling, locale-agnostic
│   │   └── webhooks/
│   │       ├── stripe/route.ts
│   │       └── docuseal/route.ts
│   ├── actions/                 # Server Actions, organized by surface — locale-agnostic
│   │   ├── leads.ts
│   │   ├── proposals.ts
│   │   └── ...
│   ├── globals.css
│   └── layout.tsx               # Root layout — minimal, delegates to [locale]/layout.tsx
├── messages/
│   ├── en.json
│   └── fr-CH.json               # Default locale
├── lib/                         # App-specific utilities (not shared)
└── public/                      # Static assets
```

### Locale and route groups

Locale is **path-based**: every user-facing route lives under `app/[locale]/`, e.g. `app/[locale]/(advisor)/leads/page.tsx`, served at `/fr-CH/leads` or `/en/leads`. Path-based (not cookie- or header-based) for SEO and shareability — a lead can send a colleague a `/fr-CH/...` link and it renders in French.

`api/` and `actions/` sit *outside* `[locale]/` — they're locale-agnostic (webhooks and mutations don't have a UI locale).

Route groups, nested under `[locale]/`:

- `(public)` — unauthenticated. Landing, marketing, signup entry, auth callbacks.
- `(advisor)` — authenticated as advisor. AppShell layout (sidebar + topbar + mobile nav).
- `(client)` — authenticated as client. Client portal layout. Lands when IAF-37+ ships.

The parens are syntactic; the URL is unaffected. We use them to keep distinct layouts and access patterns isolated without polluting URLs.

### Co-located client components

Every route folder may contain `_components/`. Files there are scoped to that route. Shared primitives live in `packages/ui/components/`.

The leading underscore `_components` is conventional in Next.js — it tells the App Router this folder is private (not a route).

---

## Server Components vs Client Components

### When to use Server Component (default)

- Anything that reads data (queries from `packages/db/queries/`).
- Anything that performs auth-aware logic.
- Anything that's a layout, page, or loading boundary.
- Anything purely presentational that doesn't need browser-only APIs.

### When to use Client Component (`"use client"` directive at the top)

- Components with `useState`, `useReducer`, `useEffect`, `useRef`, etc.
- Form inputs that need controlled state, validation in real-time, or browser focus APIs.
- Components that bind to browser-only APIs (`window`, `localStorage`, `IntersectionObserver`, `navigator`).
- Components that use third-party client libraries that only run in the browser (Cal.com embed, LiveKit room).

### The "minimal client" principle

When a section needs interactivity, isolate the client part as small as possible. Pass server-rendered children into a client wrapper rather than making the whole subtree client-side.

```tsx
// page.tsx (Server Component)
import { ClientyBit } from "./_components/ClientyBit";
import { ServerOnlyData } from "./_components/ServerOnlyData";

export default async function Page() {
  const data = await getData();
  return (
    <ClientyBit>
      <ServerOnlyData data={data} />
    </ClientyBit>
  );
}
```

`ClientyBit` carries the interactivity; `ServerOnlyData` is rendered on the server and passed as a child. The bundle stays small.

---

## Server Actions

Server Actions are the mutation API. They cross the client/server boundary, handle auth via the existing session, and emit audit events when they touch compliance-relevant tables.

### Where they live

`apps/web/app/actions/<surface>.ts`. Each file is `"use server"` at the top.

### The shape

See `server-actions` skill (Kai's domain) for the canonical pattern. From Mira's side, the call site looks like:

```tsx
// _components/AcceptLeadButton.tsx
"use client";

import { acceptLead } from "@/apps/web/app/actions/leads";

export function AcceptLeadButton({ leadId }: { leadId: string }) {
  return (
    <form action={async () => {
      const { error } = await acceptLead({ leadId });
      if (error) toast.error(error.code);
    }}>
      <button className="btn-gold">Accept lead</button>
    </form>
  );
}
```

The `form action={...}` pattern is preferred over `onClick` for mutations — it works with progressive enhancement and respects the server-first model.

### When NOT to use a Server Action

- Inbound webhooks (use `app/api/webhooks/<provider>/route.ts`).
- Long-running work (offload to a Supabase Edge Function and poll, or use streaming).
- AI calls with user-perceived latency >1s (use streaming Server Components or a dedicated streaming pattern; Vera owns those).

---

## Routes you'll write

### Page (Server Component, async)

```tsx
// app/[locale]/(advisor)/leads/page.tsx
import { listLeadsForAdvisor } from "@/packages/db/queries/leads";
import { LeadsTable } from "./_components/LeadsTable";

export default async function LeadsPage() {
  const { data, error } = await listLeadsForAdvisor();
  if (error) throw new Error(error.code);   // bubbles to error.tsx
  return <LeadsTable leads={data ?? []} />;
}
```

### Loading boundary

```tsx
// app/[locale]/(advisor)/leads/loading.tsx
import { LeadsTableSkeleton } from "./_components/LeadsTableSkeleton";

export default function LeadsLoading() {
  return <LeadsTableSkeleton />;
}
```

Skeletons match the layout footprint of the loaded state — no spinner-on-blank.

### Error boundary

```tsx
// app/[locale]/(advisor)/leads/error.tsx
"use client";
import { logger } from "@/packages/lib/logger";

export default function LeadsError({ error, reset }: { error: Error; reset: () => void }) {
  logger.error("leads_page_error", { message: error.message });
  return <ErrorEmptyState onRetry={reset} />;
}
```

Error boundaries are always Client Components (Next.js requirement), they always log to the structured logger, they always offer a `reset` action.

### Layout (Server Component)

```tsx
// app/[locale]/(advisor)/layout.tsx
import { AppShell } from "@/packages/ui/components/AppShell";
import { getAdvisorContext } from "@/packages/db/queries/advisor";

export default async function AdvisorLayout({ children }: { children: React.ReactNode }) {
  const { data: ctx } = await getAdvisorContext();
  return <AppShell context={ctx}>{children}</AppShell>;
}
```

---

## Internationalization

`next-intl` is the locale framework. Three rules:

1. **Locale is in the URL.** `app/[locale]/...` — `fr-CH` is the default.
2. **All user-facing strings come from `messages/<locale>/<surface>.json`.** No exceptions; the lint rule will catch you.
3. **Plurals and gender** use ICU MessageFormat. Date and currency formatting use `next-intl`'s `useFormatter` so they respect the locale.

Mira's surface in IAF-30 sets up scaffolding; every subsequent UI ticket adds keys to the relevant surface's JSON.

---

## Performance defaults

Phase 1 doesn't chase performance, but a few defaults stay free:

- `next/font` for Cormorant Garamond + Inter (no FOIT/FOUT, embedded into the build).
- `next/image` for any non-decorative image.
- Dynamic imports for components only used after interaction (`Modal`, heavy charts).
- Server Components stream by default; don't undo this with eager client wrappers.

Lighthouse target on public pages: Performance ≥95, Accessibility ≥95. Anything below that on a `(public)` route requires investigation before merge.

---

## Tailwind conventions

Inline utility classes. Custom utilities live in `packages/ui/styles/tailwind-preset.ts`. The preset reads from CSS variables in `packages/ui/tokens` — that's what gives us the navy/gold palette, gold-gradient, hairline borders, glass cards, etc.

Tailwind classes do *not* compose with `cn()` from a 3rd-party utility. Use a tiny local `cn` from `packages/lib/cn.ts` (clsx + tailwind-merge) and stick with it.

```tsx
import { cn } from "@/packages/lib/cn";

export function Button({ className, ...props }: ButtonProps) {
  return <button className={cn("btn-gold rounded-lg px-4 py-2", className)} {...props} />;
}
```

`@apply` is allowed only inside `packages/ui/styles/*.css` for components that genuinely benefit from class extraction (e.g., the `.card` utility). Component code never uses `@apply`.

---

## Anti-patterns we don't ship

- **`use client` at the page level.** A page is a Server Component. Move interactivity into a child.
- **Calling Supabase directly from a client component.** Always go through Server Actions or RSC-fetched data.
- **`useEffect` for data fetching.** RSC fetches data on the server. `useEffect` for browser-only side effects (focus, scroll, mutation observers), nothing else.
- **State libraries (Zustand, Redux, Jotai) for cross-route state.** URL search params and Server Components carry state across routes; in-page state is `useState`.
- **`getServerSideProps` / `getStaticProps`.** These are Pages Router; we're on App Router only.
- **`pages/` directory.** None. All routes live under `app/`.
- **Hard-coded user-facing strings.** Even for "throwaway" admin tools.
- **`console.log("debugging")`.** The structured logger or nothing.
- **Loading skeletons that don't match the loaded layout.** Skeletons that resize on data-arrival are jarring.
- **Animation libraries beyond Framer Motion.** Locked.

---

## What "done" looks like for a UI ticket

Every UI PR description includes:

1. The route(s) / component(s) shipped.
2. Link to Lior's spec comment (visible-UI tickets).
3. A screenshot or 30-second Loom of the visible change.
4. Lighthouse delta if it's a `(public)` page.
5. Any Vitest snapshots updated.

If the PR is `user-flow`-tagged, Ren reviews. Otherwise, Lior + Aria review and the PR can ship on Aria's approve once Lior signs off on the visual.

---

## Worked example — IAF-2 monorepo scaffold

The audition ticket. A folder layout sketch you'd post on the ticket as the implementation plan:

```
freedom-portal/
├── apps/web/                       # Next.js 15 App Router
│   ├── app/
│   ├── messages/
│   ├── lib/
│   └── package.json
├── packages/
│   ├── ui/                         # Design system + primitives
│   │   ├── components/
│   │   ├── tokens/
│   │   ├── styles/
│   │   └── package.json
│   ├── db/                         # Supabase types + clients (Kai)
│   ├── ai/                         # Anthropic services (Vera)
│   ├── integrations/               # Stripe, Resend, etc. (Sasha)
│   └── lib/                        # Cross-cutting utilities (logger, cn, etc.)
├── supabase/                       # Kai's primary domain
├── e2e/                            # Playwright (Ren)
├── docs/
│   ├── adr/
│   └── reference/
│       └── freedom-portal.html
├── .github/workflows/
│   ├── ci.yml                      # lint + typecheck + vitest + playwright per PR
│   └── deploy-preview.yml          # Vercel preview deploys
├── biome.json                      # lint + format
├── turbo.json                      # task orchestration
├── pnpm-workspace.yaml
├── package.json
└── tsconfig.json                   # base, extended per package
```

Verify before merge:
- `pnpm dev` boots `apps/web` on `http://localhost:3000` and renders an empty Next.js page.
- `pnpm lint` passes (Biome).
- `pnpm typecheck` passes.
- `pnpm test` runs Vitest (will be empty initially).
- `pnpm test:e2e` runs Playwright (will be empty initially).
- CI on a sample PR runs all four.

That's the audition. Keep it boring; downstream tickets will fill it.
