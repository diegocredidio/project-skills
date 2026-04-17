---
name: design-grill
description: Interrogates the user about aesthetic direction, target feel, and visual priorities until every design branch has a resolved decision. Runs only on Path B (when no DESIGN_BRIEF.md from pm-handoff exists) — otherwise design-brief-intake takes over directly. Use when user says "design grill", "interrogate the design", "help me figure out the visual direction", or when design-flow invokes it on Path B. Not for gap-filling an existing brief (that's design-brief-intake) or for post-build critique (that's design-review).
---

# Design Grill — Aesthetic Direction Interrogation

Establishes the visual and interaction direction for a feature when no prior brief exists. Produces `.design/<feature>/DESIGN_GRILL.md` that feeds `design-brief-intake`.

## When to use this (vs skip)

- If `.design/<feature>/DESIGN_BRIEF.md` exists → **skip this skill**, go to `design-brief-intake`.
- If starting from a vague idea → run this skill first.

## How to run

### Step 1: Get the subject

If the user hasn't named the feature yet, ask: "What are we designing? A sentence is enough to start."

### Step 2: Audit before asking

Before grilling, quickly scan the codebase for:
- Existing style system (`tailwind.config.*`, `globals.css`, `theme.ts`)
- Existing component libraries (shadcn, Radix, Chakra, MUI)
- Existing fonts and breakpoints
- Dark-mode setup

This lets you suggest defaults that match reality: "I see you have shadcn + Tailwind v3 and dark-mode wired with next-themes — should we build on top of that?"

### Step 3: Grill along five axes

Ask one question at a time. Wait for each answer. Probe if vague. Offer a default when the user is stuck.

**Axis 1 — Problem framing**
- What outcome does the user achieve with this feature?
- How do they do it today (if at all)?
- What's the emotional state at start and end?

**Axis 2 — Target user, emotional**
- Technical level: comfortable with dense UIs, or needs hand-holding?
- Context of use: focused desktop session, quick mobile tap, shared screen?
- What makes them drop off today?

**Axis 3 — Tone and density**
- Playful or restrained?
- Dense (information-rich) or spacious (breathing room)?
- Warm or cool palette?
- Serif, sans, or mono for headlines?

**Axis 4 — References and anti-references**
- Show me 2-3 products you admire for this kind of feature.
- Show me 2-3 products that make this feature wrong.
- What's the one adjective you want someone to use after 10 seconds?

**Axis 5 — Constraints**
- Must support dark mode? (Default: yes, dark-first.)
- Mobile priority? (Default: mobile-first baseline 375px.)
- Accessibility target? (Default: WCAG AA.)
- Any brand requirements already locked?

### Step 4: Offer defaults on everything

When the user says "I don't know", offer a specific default and move on:

- **Dark mode:** yes, dark-first with light as variation
- **Breakpoints:** 375 / 768 / 1024 / 1440
- **Component baseline:** shadcn + Radix primitives (if codebase supports it)
- **Density:** spacious, 4px spacing unit
- **Motion:** subtle — 150ms ease-out on hover, 250ms on layout shifts

### Step 5: Surface contradictions immediately

If the user says "minimal and restrained" but then asks for "bold, attention-grabbing CTAs", call it out: "Those two will fight each other — which wins when they conflict?"

### Step 6: Produce DESIGN_GRILL.md

Save to `.design/<feature>/DESIGN_GRILL.md`:

```markdown
# Design Grill: [Feature Name]

**Date:** [today]
**Mode:** scratch

## Problem framing
[2-3 sentences]

## Target users (emotional)
[Level, context, dropoff]

## Tone and density
- Tone: [descriptor]
- Density: [spacious / moderate / dense]
- Palette temperature: [warm / neutral / cool]
- Typography style: [serif / sans / mono / mixed]

## References and anti-references
- Admire: [product, why]
- Avoid: [product, why]
- Target adjective: [one word]

## Constraints locked
- Dark mode: [yes/no]
- Breakpoints: [list]
- A11y target: [level]
- Brand: [constraints if any]

## Codebase reality (from quick audit)
- Style system: [...]
- Component library: [...]
- Dark mode setup: [...]

## Open questions
[What couldn't be resolved — flag for design-brief-intake to revisit]
```

### Step 7: Hand off

Tell the user: "Direction recorded. Next: `design-brief-intake` will build the canonical brief and audit the codebase in depth."

## Rules

- One question at a time. Never drop a three-question block.
- Never accept "whatever looks good" — offer a default and confirm.
- If the user struggles with abstract adjectives, ask for concrete product references instead.
- Don't invent a filosofia estética catalog ("Dieter Rams", "Brutalist") — let the user describe it in their own words. Canned styles produce caricatures.
- The grill should produce a decision for every axis — "deferred" is an explicit option, but it must be chosen, not skipped.
