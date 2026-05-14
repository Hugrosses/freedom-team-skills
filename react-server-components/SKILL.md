---
name: react-server-components
description: Use this skill before deciding whether a component is Server or Client, before adding "use client", and before approving any PR that crosses the server/client boundary. Triggers on every component with state, every page, every layout.
---

# React Server Components — Freedom's Server-First Posture

> Stub. Full skill to be authored when Mira ships IAF-7 (app shell) — that's the first ticket where the server/client split has real architectural consequences (sidebar nav state, topbar interactivity, server-rendered data).

## The non-negotiables

- **Server Components by default.** Every page, every layout, every read-only display lands on the server unless there's a concrete reason to move it.
- **`"use client"` is a signal.** Each instance is justified by the comment in the PR description ("needs `useState` for the modal", "needs `useRef` for focus", etc.).
- **Server data flows down as props.** Don't fetch client-side; fetch on the server, hand the result to a Client Component as a prop.
- **No `useEffect` for data fetching.** That's a code smell that says "this should be a Server Component."
- **Streaming is free.** RSC streams by default; don't break this with eager `await Promise.all` for everything.
- **Suspense boundaries match Loading boundaries.** `loading.tsx` per route segment; `<Suspense>` for finer-grained streaming.

## Patterns to author here later

- The "minimal client" pattern: pass server-rendered children into a thin client wrapper.
- Composing Server and Client Components — the `children` pattern, the import boundaries.
- When to reach for Server Actions vs server-rendered forms.
- Streaming patterns for AI-generated content (Vera coordinates).
- Anti-patterns: `"use client"` at the page level, importing server-only utilities into client files, leaking secrets through props.
- Testing: how to test a Server Component (snapshot of rendered output) vs a Client Component (Vitest with @testing-library/react).

Author in full when IAF-7 ships.
