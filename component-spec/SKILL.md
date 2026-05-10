---
name: component-spec
description: Use this skill before writing any component spec, before naming a new component, and before approving a Mira PR that ships a new primitive. Triggers on every UI ticket where a component doesn't already exist.
---

# Component Spec — Freedom's UI Documentation Format

> Stub. Full skill to be authored when Lior produces the second or third real spec — at that point we know which sections of the format are load-bearing and which are theater. The format below is the starting template (also embedded in `agents/lior/instructions.md` section 8).

## The non-negotiables

- **Every new component gets a spec before it gets implemented.** Mira does not invent a component from a ticket title; she reads Lior's spec.
- **Spec lives as a comment on the ticket.** Not in a separate doc, not in a Notion page. Co-located with the work.
- **Spec covers all states.** Default, hover, active, disabled, loading, error. Missing states default to "matches default" only when explicitly stated.
- **Mockup line numbers anchor the spec.** When the mockup has a reference instance, point at it: `docs/reference/freedom-portal.html` lines NN–MM. Saves both Lior and Mira from guessing.
- **Component names are nouns.** `LeadScoreCard`, not `RenderLeadScore`. PascalCase. Descriptive. Lior owns the naming vocabulary.
- **Accessibility expectations stated, not assumed.** ARIA role, keyboard interaction, focus order, contrast ratio against the background.

## The format

```markdown
# <ComponentName>

## Purpose
<one sentence — what user need does this serve>

## Mockup reference
- File: docs/reference/freedom-portal.html
- Lines: NN–MM

## Anatomy
- <part>: <token / size / behavior>

## States
- Default | Hover | Active | Disabled | Loading | Error

## Motion
- Property | Easing | Duration

## Copy
- EN | FR-CH

## Accessibility
- Keyboard | ARIA | Contrast

## Open questions
<flag explicitly>
```

## Patterns to author here later

- The first 5 component specs Lior produces, indexed here as worked examples.
- The naming taxonomy (when to call something a Card vs a Panel vs a Tile vs a Surface).
- The "compose vs new" decision — when a ticket warrants a new primitive vs composing existing ones.
- Spec evolution: how to revise a spec after Mira's first implementation reveals constraints, without breaking the audit trail.

Author in full once the first 3 specs are in the wild.
