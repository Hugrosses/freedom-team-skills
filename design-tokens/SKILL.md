---
name: design-tokens
description: Use this skill before adding any color / typography / spacing / motion value, before approving any PR that introduces a new visual constant, and before writing the design-tokens spec for IAF-3. Triggers on every "what hex / what spacing / what easing" question.
---

# Design Tokens — Freedom's Visual Vocabulary

This is the source of truth for *what* visual values exist and *when* to use which. Mira ports the values into a Tailwind preset; you (Lior) decide which values exist in the first place.

The system is deliberately small. Calm UI emerges from a tight palette and a small spacing scale, not from creative variation. If you find yourself wanting to add a token "just for this one case," resist — that's how systems drift into Salesforce-shape.

---

## The non-negotiables

1. **Tokens live as CSS variables** in `packages/ui/tokens/tokens.css`. Tailwind reads them via `packages/ui/styles/tailwind-preset.ts`.
2. **Every visible value in code references a token.** No literal hex, no literal `12px` outside the token file. The lint rule (when it exists) catches this; until then, reviewers do.
3. **New tokens require a spec entry** in this skill (or in the IAF ticket that introduces the token). No silent additions.
4. **Locked tokens** (color, typography) require Hugo's approval to change. **Extensible tokens** (spacing, motion) can grow with team consensus.
5. **Tokens are semantic, not decorative.** `--gold` is the brand accent — never used for "just a yellow." If you want a yellow, you don't have one; you have gold or you have the wrong instinct.

---

## Color (locked)

Ported from `docs/reference/freedom-portal.html` lines 24–41. These are the canonical values; do not adjust without a Hugo conversation.

### Navy scale (backgrounds, surfaces)

| Token | Hex | Used for |
|---|---|---|
| `--navy-900` | `#070e1d` | App background, body |
| `--navy-800` | `#0a1628` | Topbar, panel base |
| `--navy-700` | `#101f38` | Card base 2nd-tier |
| `--navy-600` | `#162845` | Card hover, glass overlay top |
| `--navy-500` | `#1e3356` | Mid-tone surface, scrollbar thumb |

Pick darker for the page chrome, lighter for elevated surfaces. The five-step scale is enough; you should not feel a need for `--navy-650`.

### Gold scale (accent, brand identity)

| Token | Hex | Used for |
|---|---|---|
| `--gold` | `#d4af37` | Primary accent — CTAs, active nav, signature |
| `--gold-soft` | `#e7c96a` | Secondary accent — chip text, gold-text fade endpoint |
| `--gold-hi` | `#f0d069` | Top of `gold-gradient`, hover highlight |
| `--gold-dk` | `#9d7f1f` | Bottom of `gold-gradient`, embossed depth |
| `--line` | `rgba(212,175,55,.14)` | Hairline borders, dividers (calm) |
| `--line-strong` | `rgba(212,175,55,.28)` | Hairline borders, dividers (assertive) |

Gold is the brand. Use it deliberately. A page with three gold elements is luxurious; a page with thirty is a casino.

### Cream + muted (text)

| Token | Hex | Used for |
|---|---|---|
| `--cream` | `#f5f1e8` | Primary body text on navy backgrounds |
| `--muted` | `#a8b3c7` | Secondary body text, sub-headlines |
| `--muted-2` | `#6c7891` | Tertiary text, captions, table headers |

Three text shades. That's it. If you want "almost-cream-but-slightly-warmer" you don't.

### State colors (limited)

| Token | Hex | Used for |
|---|---|---|
| `--green` | `#3ecf8e` | Success badges, positive deltas, "signed" / "joined" / "active" states |
| `--red` | `#ef6060` | Error badges, negative deltas, "failed" states |

Yellow / orange / blue do not have tokens. Lior signs off if a state genuinely needs a fourth color, and only via an ADR.

---

## Typography (locked)

Two families, loaded via `next/font`:

- **Cormorant Garamond** — serif. Weights 400, 500, 600, 700. Use for: headings (h1–h3), brand sigils, signature flourishes (the e-sign Cormorant flourish, the cover-letter greeting), the "Freedom" wordmark when on a navy background.
- **Inter** — sans. Weights 300, 400, 500, 600, 700. Use for: body, UI copy, buttons, chips, table data, monospace fall-back when `mono` not specified.

### Type scale (extensible — add when needed)

| Token | Size | Line-height | Used for |
|---|---|---|---|
| `--text-xs` | 11–12px | 1.25 | Microcopy, table headers, badge text |
| `--text-sm` | 13.5–14px | 1.4 | Body, UI labels, table cells |
| `--text-base` | 15px | 1.5 | Long-form body |
| `--text-lg` | 18px | 1.4 | Card titles |
| `--text-xl` | 22px | 1.3 | Section heads |
| `--text-2xl` | 28px | 1.2 | Page heads |
| `--text-hero` | 48–56px | 1.05 | Landing hero, marketing |

Letter-spacing: `-0.01em` on Cormorant headings (`.serif` utility); default for Inter.

Headings always use Cormorant. UI surface text always uses Inter. The mix is the brand signature; don't muddle it.

### Tabular numerics

For aligned monetary values and ratios in tables, use the `.mono` utility (`ui-monospace, SFMono-Regular, Menlo`). Don't reach for Cormorant numerals in dashboards.

---

## Spacing (extensible)

Tailwind's default scale is fine but Freedom uses a *narrower* effective range:

- **Inside a card / row:** 4 / 8 / 12 / 14 / 16 / 18 / 24 (px-equivalents `1 1.5 2 3 3.5 4 4.5 6` in Tailwind units).
- **Between cards / sections:** 24 / 32 / 48 / 64 (px). Larger gaps are reserved for landing-style hero sections.
- **Layout grid:** 12-column on desktop, 4-column on mobile. Gutter 16–24px.

Avoid 5px / 7px / 11px increments. The rhythm comes from staying on the scale.

---

## Border radius (locked)

| Token | Value | Used for |
|---|---|---|
| `--radius-sm` | 6px | `kbd`, badges, chips |
| `--radius-md` | 10px | Inputs, buttons |
| `--radius-lg` | 14px | Toasts, smaller cards |
| `--radius-xl` | 18px | Cards, primary panels |
| `--radius-2xl` | 22px | Modals, hero panels |

`radius-full` (`999px`) for pill chips and avatars only. Avoid arbitrary `border-radius: 7px` or `rounded-[5px]`.

---

## Elevation (locked)

We use **two** elevation patterns. Resist the urge for a third.

1. **Hairline border + glass background.** `.glass` utility: gradient navy with `backdrop-filter: blur(12px)` + `border: 1px solid var(--line)`. This is the default surface.
2. **Glow gold.** `.glow-gold` utility: soft gold rim + outer drop-shadow. Used sparingly — featured plan card, the "you're set" finalize moment, the signed-proposal celebration.

No `box-shadow: 0 1px 2px rgba(0,0,0,0.05)`. Material-design-style elevation breaks the brand.

---

## Motion (extensible)

Motion language is calm: short durations, subtle easing, no bouncing. Three duration tokens, three easing tokens, in that combination.

### Durations

| Token | ms | Used for |
|---|---|---|
| `--motion-fast` | 150 | Hover state, focus ring, button press |
| `--motion-base` | 250 | Modal open, fade-in, panel slide |
| `--motion-slow` | 400 | Page transitions (rare), animated charts |

Anything longer is a celebration moment (the confetti at signing, the compliance-review animation). Don't reach for slow durations as a default.

### Easings

| Token | Curve | Used for |
|---|---|---|
| `--ease-out` | `cubic-bezier(.2, .9, .3, 1)` | Default — appears, settles |
| `--ease-in-out` | `cubic-bezier(.4, 0, .2, 1)` | State transitions both directions |
| `--ease-spring` | `cubic-bezier(.34, 1.56, .64, 1)` | Reserved for the celebration moments only |

No ease-linear. No abrupt cuts. Motion respects the brand's "calm" descriptor.

### Forbidden

- Bouncing buttons.
- Shimmer skeletons (use the existing fade-in pattern).
- Auto-playing carousels.
- Parallax on scroll.

---

## Iconography (locked)

**Lucide** is the only icon set. Loaded via the Lucide CDN in the mockup; via `lucide-react` in code.

Rules:

- Stroke width: default Lucide (`1.5`).
- Sizes: `w-3.5 h-3.5` (small), `w-4 h-4` (medium, default), `w-5 h-5` (large), `w-7 h-7` (hero / KPI cards). Don't invent new sizes.
- Color: inherit from text (`color: var(--cream)` body, `color: var(--gold)` accent, `color: var(--muted)` secondary).
- No emoji in user-facing UI. Even error states. A Lucide icon or nothing.

If a needed icon isn't in Lucide, ask Aria; we don't mix icon sets.

---

## Component-specific token applications

These aren't new tokens; they're vetted *combinations* of the tokens above. Codified here so Mira doesn't reinvent per component.

### `.card`

Background: gradient `rgba(22,40,69,.8)` → `rgba(16,31,56,.8)`.
Border: `1px solid var(--line)`.
Radius: `var(--radius-xl)` (18px).
Hover: border becomes `var(--line-strong)`, lifts `translateY(-2px)`, transition `all 200ms ease-out`.

### `.btn-gold`

Background: `linear-gradient(180deg, var(--gold-hi), var(--gold))`.
Color: `var(--navy-800)` (`#0a1628`).
Font: Inter 600.
Hover: `translateY(-1px)`, `filter: brightness(1.05)`, gold drop-shadow.

### `.chip`

Background: `rgba(212,175,55,.06)`.
Border: `1px solid var(--line-strong)`.
Color: `var(--gold-soft)`.
Padding: 4 / 10 (px). Radius: full.

### `.hairline`

Border-top: `1px solid var(--line)`. Used between sections within a card.

### `.divider`

Background: `linear-gradient(90deg, transparent, var(--line-strong), transparent)`. Height 1px. The "branded hr."

---

## Adding a token

When you genuinely need a new value:

1. **Confirm it can't be expressed with existing tokens.** Most "new" needs are someone wanting `--gold-soft` instead of `--gold`.
2. **If color or typography:** ping Hugo via Aria. These are locked.
3. **If spacing / motion / radius:** propose in the PR, get Mira's lift, ship in the token file with a note.
4. **Update this skill** in the same PR. The skill stays the source of truth.
5. **Update the Tailwind preset** so `bg-<token>` / `text-<token>` / `rounded-<token>` work.

Anti-pattern: adding a token in `tokens.css` without updating this skill or the preset. Future you will find a hex with no provenance and have to guess.

---

## What "done" looks like for IAF-3

The audition ticket. Lior's spec, posted as a comment on IAF-3, contains:

1. The full token table (color, typography, spacing, radius, motion, iconography) — copy-pasted from this skill, with any project-specific deltas explicitly called out (there shouldn't be any for phase 1).
2. The component-specific applications above (`.card`, `.btn-gold`, `.chip`, `.hairline`, `.divider`).
3. Sample renders: a card with a button and a chip, on the navy background, in Cormorant + Inter — link to a screenshot or a Codepen if you can. Otherwise an inline ASCII / annotated description that Mira can implement against.
4. A note for Mira on the Tailwind preset: which utilities to expose (`bg-navy-{900..500}`, `text-cream`, `text-muted`, `text-muted-2`, `text-gold`, `text-gold-soft`, `border-line`, `border-line-strong`, `rounded-{sm..2xl}`, `font-serif`, `font-mono`, plus the composite `glass` / `card` / `btn-gold` / `chip` / `hairline` / `divider` utilities).
5. Open questions, if any. Surface them — don't bury.

That spec lets Mira ship IAF-3 in a single sitting. That's the ergonomic target.
