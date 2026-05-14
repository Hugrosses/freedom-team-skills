---
name: framer-motion
description: Use this skill before adding any animation, before reaching for a third-party motion library, and before approving a PR that introduces movement. Triggers on every fade, slide, hover state, modal open, and "should this animate?" question.
---

# Framer Motion — Freedom's Motion Library

> Stub. Full skill to be authored when motion lands in earnest — likely IAF-7 (app shell with sidebar slide / mobile-nav transition) or IAF-10 (onboarding step transitions).

## The non-negotiables

- **Framer Motion is the only animation library.** No GSAP, no Lottie (yet — phase 2 might revisit for the celebration moment), no anime.js. Locked in `tech-stack-canon`.
- **Calm, not bouncy.** Default easing is `cubic-bezier(.2, .9, .3, 1)` (ease-out). No spring physics by default. The brand's "calm UI" descriptor is your guardrail.
- **Motion tokens from `design-tokens` skill.** Duration: `--motion-fast` / `--motion-base` / `--motion-slow`. Easing: `--ease-out` / `--ease-in-out` / `--ease-spring`. Don't import arbitrary durations.
- **Respect `prefers-reduced-motion`.** Every animation has a fallback to instant when the user's OS is set to reduce motion.
- **AnimatePresence for unmounting transitions.** Don't fade out via class toggling — use the library properly.
- **Layout animations (`layout` prop) are powerful but heavy.** Use them sparingly; don't make every reorder fancy.

## Patterns to author here later

- The handful of motion presets used across the app (fade-in, slide-up, modal-open, toast-enter, page-transition).
- The celebration patterns: the confetti at signing, the compliance-review animation, the "you're set" pulse. These are *exceptional* — `--ease-spring` is allowed here.
- Performance: when a motion is expensive enough to need `will-change` / `transform: translateZ(0)`.
- Testing: how to disable motion in Playwright tests so visual regressions don't catch stride.
- Accessibility: the `prefers-reduced-motion` fallback wrapper.

Author in full when motion shows up in 3+ tickets and we have real patterns to crystallize.
