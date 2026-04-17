---
name: design-brief-intake
description: Reads the DESIGN_BRIEF.md produced by pm-handoff (or DESIGN_GRILL.md from design-grill on Path B) and enriches it with a codebase audit — existing tokens, components, fonts, breakpoints, dark-mode setup. Produces the canonical `.design/<feature>/DESIGN_BRIEF.md` (enriched) plus `CODEBASE_AUDIT.md` as separate evidence. Use when design-flow invokes it or when user says "expand the design brief", "load the design context". Not for free-form writing or for re-grilling decisions already resolved upstream.
---

# Design Brief Intake — Brief + Codebase Audit

Turns an incoming brief into a codebase-aware starting point for every downstream design skill.

## Mode detection

Check which input file exists:

- **`.design/<feature>/DESIGN_BRIEF.md` exists (pm-handoff path)** → enrich-mode
- **Only `.design/<feature>/DESIGN_GRILL.md` exists (scratch path)** → build-from-grill-mode
- **Neither exists** → abort. Tell the user: "I need either a DESIGN_BRIEF.md from pm-handoff or a DESIGN_GRILL.md from design-grill. Run one of those first."

## How to run

### Step 1: Load the input

Read `DESIGN_BRIEF.md` (enrich-mode) or `DESIGN_GRILL.md` (build-mode). Extract:
- Feature name, problem statement, target users
- Aesthetic direction, tone, references (if present)
- Screens / user journeys mentioned
- Known constraints

Tell the user: "Loaded [filename]. [N] screens mentioned, [M] decisions recorded. Now auditing the codebase."

### Step 2: Audit the codebase

Search for and read:
- **Style system:** `tailwind.config.*`, `globals.css`, `theme.ts`, `stitches.config.*`, any `tokens.css`/`design-tokens.json`
- **Component libraries:** `package.json` deps matching `@radix-ui/*`, `shadcn*`, `@chakra-ui/*`, `@mui/*`, `@nextui/*`, or look for `components/ui/` folders
- **Fonts:** `next/font` usage, `<link>` tags in app layout, `@font-face` in CSS
- **Dark mode:** `next-themes`, `class="dark"` pattern, CSS `@media (prefers-color-scheme)`
- **Breakpoints:** declared in Tailwind config, CSS container queries, styled-system config
- **Routing:** `app/`, `pages/`, `src/routes/` tree to understand current IA

Record findings as a structured inventory.

### Step 3: Gap analysis

Compare what the brief declares against what the codebase already has. Classify every design concern as:

- ✅ **Clear** — brief and codebase agree
- ⚠️ **Ambiguous** — brief is vague or codebase differs
- ❌ **Missing** — brief is silent and codebase has nothing

Surface conflicts directly (e.g., "Brief says dark-mode-first but codebase has no theme toggle").

### Step 4: Enrich the brief

Write an expanded `DESIGN_BRIEF.md` that includes the original content plus these new sections:

```markdown
# Design Brief: [Feature Name]

## Project context
[From original]

## Target users
[From original]

## User journeys to design
[From original]

## Screens / pages required
[From original — may be refined based on routing audit]

## States to design (per screen)
[From original or expanded]

## Aesthetic direction
[From grill or brief — tone, density, references, anti-references]

## Existing patterns (from codebase audit)
- Style system: [Tailwind v3 / CSS modules / ...]
- Component library: [shadcn v0.9 + Radix / none / ...]
- Dark mode: [next-themes configured / none]
- Fonts: [Inter via next/font / system stack]
- Breakpoints: [sm 640 / md 768 / lg 1024 / xl 1280]

## Component inventory
| Component | Status (reuse / modify / new) | Notes |
|-----------|-------------------------------|-------|
| Button    | reuse                         | shadcn — 5 variants already defined |
| Modal     | modify                        | exists but lacks `destructive` variant |
| EmptyState| new                           | not present in codebase |

## Accessibility baseline
[WCAG level, existing a11y patterns, known gaps]

## Constraints
[Updated with codebase reality]

## Open questions
[What the audit revealed that the brief did not resolve]
```

Save to `.design/<feature>/DESIGN_BRIEF.md` (overwrite the pm-handoff version — the enriched one is canonical from now on).

### Step 5: Produce CODEBASE_AUDIT.md

Save a separate evidence file with raw audit findings, structured as:

```markdown
# Codebase Audit: [Feature Name]

**Date:** [today]

## Style system
[file paths + what was found verbatim — snippets welcome]

## Component library
[package.json deps + component inventory of existing `components/` tree]

## Fonts
[How fonts are loaded + which]

## Dark mode
[How it's configured or why it's not]

## Breakpoints
[Declared values]

## Routing / IA
[Route tree summary]

## Conflicts with brief
- [Brief said X, codebase has Y]
```

Save to `.design/<feature>/CODEBASE_AUDIT.md`.

### Step 6: Summarize and hand off

Tell the user:

> "Brief enriched. [N] decisions mapped, [M] gaps flagged, [K] conflicts surfaced. Next: `design-ia` will translate the journeys into a concrete information architecture."

## Rules

- Never overwrite user-provided content — only append/enrich structured sections.
- If the codebase audit finds a design system already in place, downstream skills must respect it (no regenerating tokens that already exist in `tailwind.config.ts`).
- Conflicts between brief and codebase are not errors — they're design decisions to surface, not hide.
- If the original brief was thin (< 20 lines), flag it: "The brief is sparse — expect design-ia and design-tokens to ask follow-up questions."
