---
name: tailwind-patterns
description: Use this skill before reaching for any Tailwind utility, before adding a custom utility to the preset, and before composing classes with cn(). Triggers on every UI ticket Mira picks up.
---

# Tailwind Patterns — Freedom's Utility Conventions

> Stub. Full skill to be authored when Mira ships IAF-3 (port design tokens + Tailwind preset) — that's the first ticket where this skill earns its keep. Until then, the rules:

## The non-negotiables

- **Inline utility classes.** No `@apply` outside `packages/ui/styles/*.css`.
- **No CSS-in-JS, no CSS Modules.** Tailwind utilities or token-CSS in `packages/ui/styles`.
- **Token-driven.** Reach for `bg-navy-800` / `text-gold` / `rounded-xl`, never `bg-[#0a1628]`. The Tailwind preset reads from `packages/ui/tokens/tokens.css`; if a value isn't in the preset, it doesn't exist yet.
- **Composite utilities for repeated patterns.** `.glass`, `.card`, `.btn-gold`, `.chip`, `.hairline`, `.divider` live in the preset; reuse them rather than re-typing the same 12 utilities.
- **`cn()` from `packages/lib/cn.ts`** (clsx + tailwind-merge) for conditional classes. Don't reach for the bigger utility libraries.
- **Mobile-first responsive.** Default styles are mobile; `md:`/`lg:` add desktop variations.
- **No arbitrary values without an open question.** `bg-[#abc]` / `mt-[7px]` is a smell — there's a missing token.

## Patterns to author here later

- The composite utility catalogue (`.glass`, `.card`, `.btn-gold`, `.chip`, `.hairline`, `.divider`) with the underlying utilities each expands to.
- The responsive breakpoints (`md` ≥ 780px per the mockup) and when to use them.
- Hover/focus/active state conventions and the focus-ring pattern.
- Anti-patterns: arbitrary values, `!important`, deep nesting via `[&>div]:`.
- The `cn()` ergonomic — when to reach for it vs string concatenation.

Author in full when IAF-3 ships — at that point we have the real preset to point at.
