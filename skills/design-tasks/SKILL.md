---
name: design-tasks
description: Reads `.design/<feature>/DESIGN_BRIEF.md`, `IA.md`, `TOKENS.md`, and `COMPONENT_SPECS.md` and produces `.design/<feature>/DESIGN_TASKS.md` — vertical-sliced, dependency-ordered design task list consumable by frontend-flow. Each task combines structure, styling, and interaction for one buildable unit. Use when design-flow invokes it or when user says "design tasks", "break down the design work". Not for generic project task generation (that's pm-tasks) or backend work breakdown.
---

# Design Tasks — Vertical-Sliced Build Plan

Turns the design plan into an ordered checklist the frontend team can execute without re-reading every spec.

## How to run

### Step 1: Load inputs

Read all four prior design artifacts:
- `.design/<feature>/DESIGN_BRIEF.md`
- `.design/<feature>/IA.md`
- `.design/<feature>/TOKENS.md`
- `.design/<feature>/COMPONENT_SPECS.md`

If any are missing, tell the user which and abort. Don't fabricate.

### Step 2: Group into phases

Organize work in 5 phases:

1. **Foundation** — tokens applied (CSS vars / Tailwind config), global layout shell, fonts loaded, dark-mode toggle wired
2. **Core screens** — one task per primary route from IA.md, built with its needed components
3. **States and interactions** — loading, empty, error, success states for each screen; motion/transitions
4. **Responsive and accessibility polish** — breakpoint sweep, keyboard nav, focus management, contrast audit
5. **Review** — hand off to `design-review` after build

### Step 3: Write tasks as vertical slices

Each task = one thin end-to-end increment. Not "build the color system, then build the spacing system" — one task does both if they travel together.

Per task, include:
- **Title** (imperative, short)
- **Description** (2-3 sentences)
- **Done when** (acceptance criteria, testable)
- **Components used** (from COMPONENT_SPECS.md)
- **Tokens used** (from TOKENS.md — key categories)
- **Depends on** (prior task IDs)
- **Estimated size** (S / M / L — 1 day / 2-3 days / 4+ days)

### Step 4: Order by dependencies

Foundation first. Then core screens in priority order (from the brief). Then states / responsive / a11y. Review last.

### Step 5: Call out parallelization

Mark tasks that can run in parallel with others (different files, no shared state). Flag for the frontend team.

### Step 6: Write DESIGN_TASKS.md

Save to `.design/<feature>/DESIGN_TASKS.md`:

```markdown
# Design Tasks: [Feature Name]

## Summary

| Phase | Count | Est. effort |
|-------|-------|-------------|
| Foundation | 3 | ~4 days |
| Core screens | 6 | ~12 days |
| States + interactions | 4 | ~5 days |
| Responsive + a11y | 3 | ~4 days |
| Review | 1 | 1 day |

## Phase 1 — Foundation

### DESIGN_TASK_001 — Apply design tokens

**Description:** Install tokens defined in TOKENS.md into the project's style system (CSS variables in globals.css, Tailwind theme extend, or theme.ts).

**Done when:**
- All semantic color tokens available in both dark and light mode
- `:root` and `.dark` selectors set
- Spacing/radius/motion tokens available via Tailwind utilities or theme object
- Visual smoke test: `Button` renders with new `bg-accent` and flips correctly on theme toggle

**Components used:** none directly (foundation)
**Tokens used:** all
**Depends on:** none
**Size:** M

### DESIGN_TASK_002 — Load fonts via next/font (or equivalent)

**Description:** ...
**Done when:** ...
**Size:** S
**Depends on:** none (parallel with 001)

### DESIGN_TASK_003 — Build page shell layout

**Description:** Implement the `app` layout from IA.md — top header (desktop), bottom nav (mobile), content area, footer.
**Done when:** layout applied to `/dashboard` route with placeholder content
**Components used:** Header, MobileNav, Footer (all new per COMPONENT_SPECS.md)
**Tokens used:** bg, surface, border, text, spacing
**Depends on:** DESIGN_TASK_001
**Size:** L

## Phase 2 — Core screens

### DESIGN_TASK_004 — Dashboard screen

**Description:** Build `/dashboard` route using PageShell layout. Hero card + recent projects list + empty state.
**Done when:**
- All three regions render with correct tokens
- Empty state shows when no projects
- Clicking a project card navigates to `/projects/[id]`
**Components used:** PageShell, Card, EmptyState, ProjectCard, Button
**Tokens used:** bg, surface, surface-hover, text, text-muted, accent, spacing, radius
**Depends on:** DESIGN_TASK_003
**Size:** M

...

## Phase 3 — States + interactions

### DESIGN_TASK_010 — Loading and error states for /projects/[id]

...

## Phase 4 — Responsive + a11y

### DESIGN_TASK_013 — Mobile breakpoint sweep

**Description:** Walk every route in IA.md at 375/428 widths, confirm nav collapses to bottom tab bar, card grids reflow to single column, modals become bottom sheets.
**Done when:** each route has a screenshot at 375 and 768 showing expected layout
**Size:** M

### DESIGN_TASK_014 — Keyboard navigation and focus management

**Description:** Tab through every interactive control; verify focus ring uses `focus-ring` token; confirm modals trap focus; confirm skip link present.
**Done when:** axe-core report shows zero focus-related violations
**Size:** M

## Phase 5 — Review

### DESIGN_TASK_016 — Design review with screenshots

**Description:** Invoke `design-review` skill with the preview URL. Capture screenshots at 375/768/1280 × (dark, light) and produce DESIGN_REVIEW.md with prioritized fix list.
**Done when:** DESIGN_REVIEW.md exists with Must/Should/Could sections and evidence screenshots
**Depends on:** all prior tasks shipped to preview
**Size:** S

## Dependency graph (text)

```
001 ─┬─ 003 ─ 004 ─┬─ 010 ─ 013 ─ 016
     │            ├─ 011
002 ─┘            └─ 012
```

## Parallelizable

- 001 ∥ 002 (different concerns)
- 004 ∥ 005 ∥ 006 (different routes, different files)
- 013 ∥ 014 (different audit dimensions)
```

### Step 7: Hand off

Tell the user: "Design tasks ordered ([N] total). Next: invoke `frontend-flow` — it reads these tasks plus TOKENS.md, IA.md, and COMPONENT_SPECS.md as implementation input. Once preview is up, come back to `design-review`."

## Rules

- Every task must have testable "Done when" criteria — "looks good" is not acceptable.
- Never mix foundation work with screen work in a single task.
- Small tasks first — large tasks get broken down until they're S or M.
- Review is always the final task, never skipped.
- Do not re-invent component specs or tokens here — only reference by name.
