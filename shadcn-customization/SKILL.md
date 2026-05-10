---
name: shadcn-customization
description: Use this skill before pulling any shadcn/ui primitive into packages/ui, before customizing one, and before approving a PR that uses a stock shadcn component. Triggers on every "let's just use shadcn" suggestion.
---

# shadcn/ui Customization — Freedom's "No Stock Defaults" Rule

> Stub. Full skill to be authored when Mira ships IAF-4 (core UI primitives) — at that point we'll have a worked example of "what customization looks like" for Button, Card, Input, etc.

## The non-negotiables

- **shadcn is a starting point, not a destination.** A component lands looking shadcn-default, Lior refuses the PR. The whole point of shadcn is that we *own* the source — the customizations are why we're using it.
- **Customize on import.** Don't ship a stock `button.tsx`; replace `bg-primary` with `btn-gold`, `text-primary-foreground` with our cream-on-navy, the focus ring with our gold-tinted ring.
- **Lior signs off on the customization** before Mira propagates it across the app. The first instance of every primitive becomes the template.
- **No shadcn theme provider.** We use CSS variables from `packages/ui/tokens` directly, not shadcn's theming layer.
- **No shadcn "blocks" (composite layouts).** Those are tied to the shadcn aesthetic. Build composites from primitives instead.

## Patterns to author here later

- The customization checklist applied to each primitive (Button, Card, Input, Textarea, Select, Badge, Chip, Toast, Modal, KPI, Divider).
- The "stock-vs-Freedom" diff for the most-customized primitives, so reviewers can spot drift fast.
- When to use shadcn at all vs. write from scratch — for highly bespoke pieces (the proposal projection chart, the kanban) shadcn isn't worth it.
- Accessibility: shadcn ships solid a11y; don't strip it during customization.

Author in full when IAF-4 ships.
