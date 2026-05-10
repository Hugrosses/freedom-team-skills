---
name: design-tokens
description: Use this skill before wiring any token into the Tailwind preset, before approving a PR that adds a token to packages/ui/tokens, and before answering "where do design values come from in Freedom?". Triggers on every token-related implementation question. For token *values* (which hex, which scale, which easing), defer to freedom-portal-designer.
---

# Design Tokens — How Tokens Are Wired Into Freedom

This skill is **implementation patterning**. It explains where tokens live in the codebase, how Tailwind reads them, the rules around adding or changing one, and how Lior's specs flow into Mira's preset.

**It does not list token values.** The canonical source of truth for *what* tokens exist (which colors, which typography scale, which spacing rhythm, which easing curves) is **`freedom-portal-designer`**, the senior product designer skill. Lior owns that source. This skill consumes it.

If you find yourself wanting to write a hex code or a px value here, stop — that belongs in `freedom-portal-designer`. This file is about plumbing.

---

## Source-of-truth chain

```
freedom-portal-designer  (Lior's senior-designer mandate, "Brand & Design DNA")
        │
        ▼  Lior produces a spec for IAF-NNN, posted as a ticket comment
design-tokens spec       (the spec format below)
        │
        ▼  Mira ports the spec into code
packages/ui/tokens/tokens.css       ← CSS variables; the runtime values
packages/ui/styles/tailwind-preset  ← reads the CSS vars, exposes utilities
        │
        ▼  Components consume utilities (or composite utilities like .glass / .card)
apps/web                          ← Mira's pages, never bare hex / px
```

Every implementation file at the bottom of this chain points back through the chain. A reviewer should be able to read a class like `bg-navy-800` and trace it back to a CSS variable, and the variable's *value* back to a line in `freedom-portal-designer` (or to a Lior-specced delta with rationale).

---

## The non-negotiables

1. **Tokens live as CSS variables** in `packages/ui/tokens/tokens.css`. Tailwind reads them via `packages/ui/styles/tailwind-preset.ts`. There is no second source.
2. **No literal hex / px values in code.** Every visible value in `apps/web` and `packages/ui/components` references a token (`bg-navy-800`, `text-gold`, `rounded-md`). Composite utilities (`glass`, `card`, `btn-gold`, `chip`, `hairline`, `divider`) live in `packages/ui/styles/*.css` and are the only place `@apply` is permitted.
3. **A token's value comes from `freedom-portal-designer`** unless Lior has posted a delta on a ticket with explicit rationale. Mira does not invent values; Mira ports them.
4. **Adding a new token is a code change AND a skill change.** Mira's PR that adds a token in `tokens.css` must be paired with either an update to `freedom-portal-designer` (Lior owns that PR) or a citation of where the value already exists in `freedom-portal-designer`.
5. **Locked vs extensible.** Color and typography are locked — Hugo signs off via `freedom-portal-designer` updates. Spacing, radius, motion, iconography are extensible — Lior approves additions, Aria is informed.

---

## Token namespaces and naming

Tokens are grouped into namespaces; each lives as a CSS variable with a predictable prefix so the Tailwind preset can map them mechanically.

| Namespace | Prefix | Example | Tailwind utility |
|---|---|---|---|
| Background / surface | `--navy-*` | `--navy-800` | `bg-navy-800` |
| Brand accent | `--gold*` | `--gold`, `--gold-soft`, `--gold-deep` | `bg-gold`, `text-gold-soft`, etc. |
| Text | `--cream`, `--muted*` | `--muted-2` | `text-muted-2` |
| Border / line | `--line`, `--line-strong` | — | `border-line` |
| State | `--success`, `--warning`, `--danger` | — | `text-danger`, `bg-success/10` |
| Typography family | `--font-serif`, `--font-sans`, `--font-mono` | — | `font-serif` |
| Type scale | `--text-*` | `--text-lg` | `text-lg` |
| Spacing | tailwind built-in (`--space-*` if we ever override) | — | `p-3`, `gap-4` |
| Radius | `--radius-*` | `--radius-md` | `rounded-md` |
| Motion duration | `--motion-*` | `--motion-base` | `duration-base` |
| Motion easing | `--ease-*` | `--ease-out` | `ease-out` |

Naming conventions:

- Lowercase, kebab-case.
- Suffix `-strong` / `-soft` / `-deep` for variants of the same hue, *not* numeric scales (we use numeric only for backgrounds: `--navy-900..500`).
- Prefix `--font-` for type families. Prefix `--text-` for sizes. Don't conflate.
- The Tailwind preset utility name should be derivable from the variable name without a lookup table.

When `freedom-portal-designer` introduces a token name, this skill mirrors the namespace; if the prefix collides with an existing namespace, that's a question for Lior (see "propose, don't assume" below).

---

## Tailwind preset shape

`packages/ui/styles/tailwind-preset.ts` reads `tokens.css` via CSS-variable references:

```ts
// Sketch — Mira finalizes during IAF-3 implementation.
import type { Config } from "tailwindcss";

export const preset: Partial<Config> = {
  theme: {
    extend: {
      colors: {
        navy: {
          900: "var(--navy-900)",
          800: "var(--navy-800)",
          700: "var(--navy-700)",
          600: "var(--navy-600)",
          500: "var(--navy-500)",
        },
        gold: {
          DEFAULT: "var(--gold)",
          soft: "var(--gold-soft)",
          deep: "var(--gold-deep)",
        },
        cream: "var(--cream)",
        muted: {
          DEFAULT: "var(--muted)",
          2: "var(--muted-2)",
        },
        line: "var(--line)",
        "line-strong": "var(--line-strong)",
        success: "var(--success)",
        warning: "var(--warning)",
        danger: "var(--danger)",
      },
      fontFamily: {
        serif: ["var(--font-serif)"],
        sans: ["var(--font-sans)"],
        mono: ["var(--font-mono)"],
      },
      fontSize: {
        // bound to --text-* tokens once Lior's spec lands
      },
      borderRadius: {
        // bound to --radius-* tokens once Lior's spec lands
      },
      transitionDuration: {
        fast: "var(--motion-fast)",
        base: "var(--motion-base)",
        slow: "var(--motion-slow)",
      },
      transitionTimingFunction: {
        out: "var(--ease-out)",
        "in-out": "var(--ease-in-out)",
        spring: "var(--ease-spring)",
      },
    },
  },
};
```

The preset names track `freedom-portal-designer` token names exactly. If `freedom-portal-designer` says `--gold-soft`, the preset exposes `text-gold-soft`. Mira does not invent shorthand.

---

## Composite utility CSS

`packages/ui/styles/composites.css` is the only file where `@apply` is permitted. It defines reusable visual signatures so Mira doesn't repeat 12 utilities per card:

```css
/* sketch — exact rules come from Lior's spec on IAF-3 and follow-ups */

.glass {
  @apply backdrop-blur-md border border-line bg-gradient-to-b from-navy-600/65 to-navy-700/55;
}

.card {
  /* Lior to specify the exact gradient + radius in the IAF-3 spec */
}

.btn-gold {
  /* Lior to specify gradient direction, hover behavior, focus ring */
}

.chip { /* … */ }
.hairline { /* … */ }
.divider { /* … */ }
```

Why composites: a "card" appears 50 times across Freedom; if every page composes 8 utilities to render one, drift is inevitable. Composites are the tax we pay to prevent drift.

When Lior adds a new composite, this file gains a new class; the skill itself does not need to re-document the composite (`freedom-portal-designer` is the source).

---

## Propose, don't assume — when `freedom-portal-designer` underspecifies

`freedom-portal-designer` is intentionally high-level. There will be moments where it underspecifies for an implementation question — for example:

- Two token names that collide ("`--gold-hi` from one source, `--gold-soft` from another, same hex").
- A scale gap (a `--text-15` you "need" between `--text-14` and `--text-16`).
- A composite utility that combines tokens whose interaction `freedom-portal-designer` doesn't address.

In every such case, **Lior proposes; nobody assumes.**

A proposal is:

1. Posted on the relevant ticket as a comment from Lior.
2. Marked explicitly: `**PROPOSAL** — fills gap in freedom-portal-designer §<section>.`
3. Cites the underlying rationale (the source line in `freedom-portal-designer` that's silent on this, plus design reasoning).
4. Requests review from Hugo + Aria. Held until both approve.
5. On approval: Lior updates `freedom-portal-designer` so the next ambiguity gets resolved at the source. Mira ports.

What is **not** acceptable:

- Mira inferring a value because "it's obviously this hex."
- Lior posting a token without rationale and treating Mira's silence as approval.
- Anyone updating `tokens.css` without the proposal trail.

Treat this skill's "propose, don't assume" rule the same way `audit-logging` treats the audit-event contract: every token decision is a row in the trail, and the trail must be complete to be useful.

---

## Adding a token

When `freedom-portal-designer` already has the value:

1. Mira opens a PR adding the variable to `tokens.css`, expanding the Tailwind preset, and (if applicable) the composite class.
2. PR description cites the line in `freedom-portal-designer` where the value comes from.
3. Lior reviews for fidelity (was the right hex picked? does the namespace fit?). Aria reviews for plumbing.

When `freedom-portal-designer` does not have the value yet:

1. Lior posts a proposal on the ticket per "Propose, don't assume" above.
2. On approval, Lior updates `freedom-portal-designer` first.
3. Then Mira's PR cites the new section. (No "land both at once" shortcut — the source-of-truth chain matters.)

---

## What "done" looks like for IAF-3

IAF-3 is split conceptually into two phases inside one ticket:

**Phase A — Lior's audition: spec authoring.**
Lior reads `freedom-portal-designer` and produces the v2 token spec for Freedom Portal as a comment on IAF-3. The spec:

1. Names every token in every namespace (color, typography, spacing, radius, motion, iconography).
2. Cites the source line(s) in `freedom-portal-designer` for each value.
3. Marks every gap (any place `freedom-portal-designer` is silent or contradictory) as a `**PROPOSAL**` with rationale, per "Propose, don't assume."
4. Provides composite utilities (`.glass`, `.card`, `.btn-gold`, `.chip`, `.hairline`, `.divider`) with the exact `@apply` recipe Mira will port.
5. Open questions surfaced explicitly, never buried.

Phase A reviewers: **Hugo + Aria**. Both must explicitly approve. No "silent acceptance after 24h."

**Phase B — Mira's port.**
Once Phase A is approved, Mira ports the spec into `packages/ui/tokens/tokens.css`, `packages/ui/styles/tailwind-preset.ts`, and `packages/ui/styles/composites.css`. PR description cites the approved spec comment.

Phase B reviewers: **Lior** (fidelity) + **Aria** (plumbing).

If Lior's spec is clean enough that Phase B is fully decoupled, Aria splits the ticket into IAF-3 (spec) and IAF-3.5 (port). Default: stays single-ticket; the split is a judgment call after the spec lands.

---

## Anti-patterns we don't ship

- **Hex / px values in `apps/web` or `packages/ui/components`.** They live in `tokens.css` only.
- **Tokens added without a citation back to `freedom-portal-designer` or a Lior proposal.** Untraceable values rot.
- **`@apply` outside `packages/ui/styles/`.** Component code uses utility classes inline.
- **A second source of token values** (a Notion doc, a Figma file, a comment on a closed PR). The chain `freedom-portal-designer` → spec → tokens.css is the only path.
- **Tailwind preset names that don't mirror the variable names.** `bg-darkBlue900` for `--navy-900` is a smell.
- **Lior approving Phase B in `freedom-portal-designer`'s name without first posting the spec on the ticket.** The trail is the contract.
