---
name: accessibility-audit
description: Use this skill before approving any user-facing UI PR, before signing off on a public surface, and before declaring a Lighthouse score acceptable. Triggers on every visible-UI ticket Lior reviews.
---

# Accessibility Audit — Freedom's WCAG Posture

> Stub. Full skill to be authored when Lior performs the first real audit — likely on IAF-8 (landing) or IAF-10 (onboarding) once Mira ships the first user-facing surface.

## The non-negotiables

- **Lighthouse Accessibility ≥95** on every public-facing route. Below that, investigate before merge.
- **Keyboard-first.** Every interactive surface is reachable and operable via keyboard. Tab order is sensible. Focus is always visible (gold focus ring, never `outline: none` without replacement).
- **Contrast ≥ 4.5:1 for body text** (WCAG AA). The cream-on-navy palette comfortably passes; check anyway when reaching for `--muted-2` against `--navy-700`.
- **ARIA used sparingly.** Native HTML semantics first; ARIA only when there's no native equivalent. Bad ARIA is worse than no ARIA.
- **Forms have labels.** Every input has either a visible `<label for="...">` or `aria-label`. No placeholder-only labels.
- **Modals trap focus.** Tab cycles within the modal; Escape closes; focus returns to the trigger on close.
- **Motion respects `prefers-reduced-motion`.** See `framer-motion`.
- **Images have alt.** Decorative images get `alt=""` (explicit empty). Informative images describe what they convey.

## Patterns to author here later

- The audit checklist applied to a real surface (landing, onboarding, lead detail) with screenshots and Lighthouse outputs.
- Common mistakes Mira makes early-on (hijacked focus rings, divs-as-buttons, missing form labels).
- When to reach for axe vs Lighthouse vs manual keyboard testing.
- Locale-specific concerns: French screen readers, RTL preparation (phase 3).
- The "compliance + accessibility" overlap — MiFID II and GDPR have a11y intersections (right to access in a usable form) that we'll touch in phase 2.

Author in full after the first audit lands.
