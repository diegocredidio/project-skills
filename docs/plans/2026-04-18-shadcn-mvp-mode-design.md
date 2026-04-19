# Shadcn MVP Mode — Design Doc

**Date:** 2026-04-18
**Status:** Approved, ready for implementation planning
**Scope:** Add a `designMode` project profile so MVP projects using shadcn/ui can run a slimmer design-flow without giving up screen design or theming decisions.

---

## Motivation

Today's `design-flow` runs the full ladder (tokens → IA → components) for every project, regardless of whether a UI framework like shadcn/ui is already in the stack. For MVP projects adopting shadcn:

- `design-tokens` re-specs a full token architecture (spacing scale, typography scale, motion, z-index) that shadcn already provides.
- `design-components` re-specs primitives (Button, Input, Dialog) that shadcn ships.

This is wasted work. But the designer still has real decisions to make: screen layouts, states, shadcn theme (colors + radius + font), custom variants, and roll-your-own components where shadcn doesn't cover.

The fix is a `designMode` flag with two values — `shadcn-theme` and `custom-system` — that a handful of skills read and behave accordingly.

Reference material for the `shadcn-theme` mode: [shadcn theming docs](https://ui.shadcn.com/docs/theming), [tweakcn](https://tweakcn.com).

---

## Decision 1 — Where the field lives and how it's captured

**New artifact:** `.pm/PROJECT_PROFILE.md`. Separate from `jira-config.md` because that file has pre-flight abort semantics (wrong scope for a design preference).

Minimum schema:

```markdown
# Project Profile
designMode: shadcn-theme | custom-system
uiFramework: shadcn/ui | <other> | none
```

**Capture in `pm-grill`:** add a new branch called **"Delivery profile"** in Full mode with a single question:

> "Modo de design: **shadcn-theme** (MVP, designer entrega tema shadcn + specs de tela) ou **custom-system** (design system completo, tokens + primitivos do zero)?"

The answer lands in two places:

1. `GRILL_SUMMARY.md` → Key Decisions Made table (for humans reading the grill).
2. `.pm/PROJECT_PROFILE.md` → canonical machine-readable source (for downstream skills).

**Gap-fill mode:** `pm-intake` scans the input document for mentions of shadcn / component library. If present, mark as ✅ Clear. If absent, add to the Critical gap list.

**Why a separate file:** downstream skills read it with a simple grep. Decoupled from artifacts that can be regenerated (PRD, ARCHITECTURE, etc.).

---

## Decision 2 — Which skills become profile-aware

Five skills + one check. Everything else stays identical.

| Skill | `shadcn-theme` behavior | `custom-system` behavior |
|---|---|---|
| `pm-handoff` | Stamps `designMode: shadcn-theme` in DESIGN_BRIEF.md and FRONTEND_BRIEF.md headers. | Stamps `custom-system`. Current behavior. |
| `design-flow` | Same step sequence. Announces "modo shadcn-theme" at Step 1 and propagates flag to sub-skills. | Current behavior. |
| `design-tokens` | Emits only shadcn theme (HSL vars + radius + font). Skips spacing/typography/motion/z-index scales. Accepts tweakcn export as input. | Current behavior (full token system). |
| `design-components` | Produces screen-centric `COMPONENT_SPECS.md`: screens + shadcn inventory + custom variants + roll-your-own. Skips primitive specs. | Current behavior (full primitive specs). |
| `frontend-components` | Reads the shadcn-mode `COMPONENT_SPECS.md` and generates a `COMPONENT_PLAN.md` that registers shadcn adds (`npx shadcn add button`) without re-specifying primitives. | Current behavior. |

**`frontend-stack` conflict check:** if `designMode = shadcn-theme` but the chosen component library ≠ shadcn/ui, abort with a reconcile prompt. Prevents silent drift.

**Unchanged:** `design-brief-intake`, `design-ia`, `design-tasks`, `design-review`, `kanban-*`.

**Filename contract preserved:** `COMPONENT_SPECS.md` stays as the filename in both modes. Only the content shape changes. Avoids breaking references in pm-handoff, frontend-components, and design-flow.

---

## Decision 3 — `TOKENS.md` format in `shadcn-theme` mode

`design-tokens` reads `.pm/PROJECT_PROFILE.md` at Step 1. If `shadcn-theme`, switch to the slim path.

**Accepted inputs (any one suffices):**

- Paste of tweakcn export (CSS block with `--background`, `--primary`, `--radius`, etc.).
- Shadcn preset name (`slate`, `zinc`, `stone`, `gray`, `neutral`, etc.) — skill resolves to the corresponding CSS.
- Manual answers to short prompts: base palette (light/dark), radius (`0 | 0.3rem | 0.5rem | 0.75rem | 1rem`), font stack.

**Output `TOKENS.md` shape:**

```markdown
# Design Tokens: [Feature] (shadcn-theme mode)

## Theme source
[tweakcn export / shadcn preset "slate" / manual]

## CSS variables
\`\`\`css
:root { --background: ...; --foreground: ...; --primary: ...; ... --radius: 0.5rem; }
.dark { --background: ...; ... }
\`\`\`

## Font stack
- Sans: Inter, system-ui
- Mono: JetBrains Mono, monospace

## What this mode does NOT include
- Full spacing scale (shadcn uses Tailwind defaults)
- Typography scale (shadcn primitives set their own)
- Motion tokens (shadcn uses Tailwind transition defaults)
- Z-index scale (shadcn components manage internally)

## Contrast verification
[AA check on primary/background pairs — flags if fail]
```

**Kept from current behavior:**

- `CODEBASE_AUDIT.md` still read. If the project already has shadcn CSS vars, don't regenerate — close gaps only.
- WCAG AA contrast check on the two critical pairs (primary/foreground, background/foreground).
- Output is markdown-only. Never writes to `globals.css` in the user's source tree.

**Removed in this mode:** current Steps 5–7 of `design-tokens` (spacing scale, typography scale, radius/motion/breakpoints/z-index tables) collapse to a single line of radius + a font stack.

---

## Decision 4 — `COMPONENT_SPECS.md` format in `shadcn-theme` mode

Same filename. Content reoriented from primitive-first to screen-first.

**Structure:**

```markdown
# Component Specs: [Feature] (shadcn-theme mode)

## Mode
shadcn-theme — primitives come from the shadcn catalog. This doc covers
screens, compositions, custom variants, and roll-your-own components.

## Shadcn component inventory (used in this feature)
| Component | Source | Customization |
|-----------|--------|---------------|
| Button    | shadcn | default variants only |
| Card      | shadcn | default |
| Dialog    | shadcn | `size=lg` variant added |
| DataTable | shadcn | sortable columns preset |

## Custom variants (shadcn component + new variant)
### Button — variant `ghost-destructive`
- Use: secondary destructive actions
- Base: shadcn Button `ghost`
- Overrides: text uses `--destructive`, hover bg `--destructive/10`
- CVA diff: [minimal snippet]

## Roll-your-own components (shadcn doesn't cover)
### ProjectCard
- Composition: Card + Badge + Avatar (all shadcn) + custom layout
- Props: `{ project: Project; onOpen: () => void }`
- States: default, hover, loading (skeleton), empty (no-cover fallback)

## Screen specs
### Screen: Dashboard (/dashboard)
- Layout: SidebarLayout (roll-your-own) > PageHeader + ProjectGrid
- Shadcn pieces: Sidebar, Breadcrumb, Input (search), Button, Card
- Custom: ProjectCard, EmptyState illustration
- States:
  - Default: 3-column grid of ProjectCards
  - Loading: 6x Skeleton cards
  - Empty: EmptyState with "Create your first project" CTA
  - Error: Alert (shadcn destructive) + retry Button

### Screen: Login (/login)
- Layout: CenteredCardLayout
- Shadcn pieces: Card, Form, Input, Button, Label, Separator
- Custom: none
- States: default, submitting (Button loading), error (FormMessage per field)
```

**Removed vs. current mode:**

- Full primitive specs (Button state tables with hex values, Input focus-ring specs).
- Token-per-component tables — becomes implicit via shadcn vars.

**Kept vs. current mode:**

- Screen-level specs with state coverage (default/loading/empty/error/success) — the designer still defines these.
- Roll-your-own components specified as today.
- Custom variants documented with a clear diff from the shadcn default.

**`frontend-components` consumption:** reads this format and emits `COMPONENT_PLAN.md` skipping every shadcn primitive (registers only the `npx shadcn add <component>` calls + the custom variants and roll-your-own specs).

---

## Decision 5 — Rollout and migration

**New projects:** `pm-grill` asks `designMode` outright. No silent default — force an intentional choice.

**Projects with existing `.pm/<feature>/` but no `PROJECT_PROFILE.md`:** `design-flow` and `pm-handoff` detect the missing file and run a one-question migration prompt:

> "Este projeto não tem PROJECT_PROFILE.md. Modo de design: shadcn-theme ou custom-system?"

Does NOT re-run `pm-grill`.

**Projects with `TOKENS.md` / `COMPONENT_SPECS.md` already generated in the legacy mode:** no auto-regeneration. The skill warns:

> "TOKENS.md existente foi gerado em custom-system. Manter, ou re-rodar em shadcn-theme? (Re-rodar substitui o arquivo — commit antes.)"

**`frontend-stack` conflict handling:** if `designMode = shadcn-theme` and the user picks a component library other than shadcn/ui, abort:

> "Conflito: PROJECT_PROFILE diz shadcn-theme mas você escolheu [X]. Reconcile antes de continuar."

**Kanban impact:** none. `kanban-sync` doesn't need to change — `designMode` affects content, not task structure. Shadcn-mode features naturally produce fewer artifacts and that shows up via `Effort`.

**README update:** add a "Project modes" section explaining both. Low priority, can ship after.

---

## Summary of files changed

| File | Change |
|---|---|
| `skills/pm-grill/SKILL.md` | Add "Delivery profile" branch (Full mode) + gap detection (Gap-fill mode). |
| `skills/pm-intake/SKILL.md` | Flag shadcn/framework mentions as Clear; absence → Critical gap. |
| `skills/pm-handoff/SKILL.md` | Read PROJECT_PROFILE; stamp `designMode` into DESIGN_BRIEF.md and FRONTEND_BRIEF.md. |
| `skills/design-flow/SKILL.md` | Announce mode at Step 1; propagate flag. |
| `skills/design-tokens/SKILL.md` | Add `shadcn-theme` branch: slim inputs + slim output; skip Steps 5–7 in that mode. |
| `skills/design-components/SKILL.md` | Add `shadcn-theme` branch producing screen-centric COMPONENT_SPECS.md. |
| `skills/frontend-stack/SKILL.md` | Add conflict check against PROJECT_PROFILE. |
| `skills/frontend-components/SKILL.md` | Honor shadcn-mode COMPONENT_SPECS; emit `npx shadcn add` plan. |
| `docs/plans/2026-04-18-shadcn-mvp-mode-design.md` | This doc. |
| `README.md` (optional, later) | "Project modes" section. |

**New files created at runtime (not in this repo):** `.pm/PROJECT_PROFILE.md` in each consumer project using the skills.

---

## Open questions

None blocking. The following are deliberately deferred:

- Additional `designMode` values beyond `shadcn-theme` / `custom-system` (e.g., `material-ui`, `chakra`) — handle only when a real project demands it.
- Automating the tweakcn import (fetching a URL, parsing the CSS) — v1 accepts a paste, which covers the case.
