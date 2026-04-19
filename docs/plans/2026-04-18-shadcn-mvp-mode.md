# Shadcn MVP Mode Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `designMode` project profile (`shadcn-theme` vs `custom-system`) that lets MVP projects using shadcn/ui run a slimmer design-flow — captured in pm-grill, read across 5 skills, with reshaped TOKENS.md and COMPONENT_SPECS.md output in shadcn mode.

**Architecture:** A single new runtime artifact (`.pm/PROJECT_PROFILE.md`) created by `pm-grill`. Downstream skills read it at their Step 1 and branch behavior. Filename contracts for `TOKENS.md` and `COMPONENT_SPECS.md` are preserved — only content shape changes. No new skills; no renames. Cross-cutting conflict check in `frontend-stack` catches drift.

**Tech Stack:** Markdown skill definitions under `skills/<skill-name>/SKILL.md`. No code, no tests — verification is `grep` against the edited file + manual reading of the new flow.

**Design doc:** `docs/plans/2026-04-18-shadcn-mvp-mode-design.md` (read it before starting).

---

## Conventions for every task

Each task edits exactly one skill file. The workflow per task is:

1. Read the current skill file (full).
2. Apply the specified edit (Edit or Write tool — prefer Edit with surgical `old_string` / `new_string`).
3. Verify with `grep` — the task lists the exact string to grep for.
4. Commit. Commit message format: `<skill-name>: add shadcn-theme mode branch` (or similar imperative).

No separate "write failing test" step — these are declarative skill specs, not executable code. The verification is that the new spec text is readable in the file and that it references `.pm/PROJECT_PROFILE.md` consistently across skills.

---

## Task 1: pm-grill — add "Delivery profile" branch + gap-fill handling

**Files:**
- Modify: `skills/pm-grill/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/pm-grill/SKILL.md`

**Step 2: Add a new branch under "Technical branches" in Step B2 (Mode B, full mode)**

Find this block in Mode B § Step B2:

```
**Technical branches:**
- Data: What data is needed? Source of truth? Ownership?
- Integrations: External systems, APIs, auth providers, payments?
- Scale: How many users? How much data? Growth trajectory?
- Constraints: Compliance? Accessibility? Performance? Budget?
- Existing codebase: What already exists that we need to respect or integrate with?
```

Replace with (add the **Delivery profile** branch at the end):

```
**Technical branches:**
- Data: What data is needed? Source of truth? Ownership?
- Integrations: External systems, APIs, auth providers, payments?
- Scale: How many users? How much data? Growth trajectory?
- Constraints: Compliance? Accessibility? Performance? Budget?
- Existing codebase: What already exists that we need to respect or integrate with?
- Delivery profile: Design mode — `shadcn-theme` (MVP, designer delivers shadcn theme + screen specs) or `custom-system` (full design system, tokens + primitives from scratch)? UI framework — `shadcn/ui`, other, or none?
```

**Step 3: Add a "Write PROJECT_PROFILE.md" step at the end of Mode B § Step B5 (before the Grill Summary template)**

Find: `### Step B5: Produce the grill summary`

Immediately after that heading and its body (which ends right before the `---` separator and the `## Grill Summary (both modes)` heading), the grill summary template already covers `GRILL_SUMMARY.md`. We need to add a parallel instruction: when the delivery profile branch is resolved, ALSO write `.pm/<feature-name>/PROJECT_PROFILE.md`.

Add a new subsection at the end of `## Grill Summary (both modes)` section, right before the `## Rules` heading:

```
---

## Project Profile (both modes)

When the Delivery profile branch is resolved, ALSO write `.pm/<feature-name>/PROJECT_PROFILE.md` — a minimal, machine-readable record that downstream skills (`pm-handoff`, `design-flow`, `design-tokens`, `design-components`, `frontend-stack`, `frontend-components`) read with a simple grep. Separate file from `GRILL_SUMMARY.md` so it survives re-runs of the grill.

Format:

\`\`\`markdown
# Project Profile
designMode: shadcn-theme | custom-system
uiFramework: shadcn/ui | <other> | none
\`\`\`

If the user didn't resolve the Delivery profile branch (gap-fill mode where the source doc pre-answered it), still write the file using the inferred values and cite the source in the GRILL_SUMMARY Key Decisions table.
```

**Step 4: Add gap-fill mode guidance**

In Mode A § Step A2 (the "Ask only the flagged questions" block), no edit needed — the INTAKE.md Grill question list already drives what gets asked. Mode A inherits correctly once `pm-intake` (Task 2) flags the Delivery profile as a gap.

But add one clarifying line at the end of Mode A § Step A4 ("Produce the grill summary"):

Find: `Write `.pm/<feature-name>/GRILL_SUMMARY.md` with ALL resolved information — both what came from the original document (already clear) and what was resolved in the grill.`

Replace with:

```
Write `.pm/<feature-name>/GRILL_SUMMARY.md` with ALL resolved information — both what came from the original document (already clear) and what was resolved in the grill. Also write `.pm/<feature-name>/PROJECT_PROFILE.md` per the "Project Profile" section below.
```

**Step 5: Verify**

Run: `Grep "Delivery profile" skills/pm-grill/SKILL.md -n`
Expected: matches in Mode B § Step B2 technical branches.

Run: `Grep "PROJECT_PROFILE.md" skills/pm-grill/SKILL.md -n`
Expected: at least 2 matches (in Mode A § Step A4 and in the new Project Profile section).

**Step 6: Commit**

```bash
git add skills/pm-grill/SKILL.md
git commit -m "pm-grill: add Delivery profile branch and PROJECT_PROFILE.md output"
```

---

## Task 2: pm-intake — detect UI framework / design mode in source documents

**Files:**
- Modify: `skills/pm-intake/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/pm-intake/SKILL.md`

**Step 2: Add a detection bullet under "Technical categories" (Step 2)**

Find this block in Step 2:

```
**Technical categories:**
- Stack or technology choices mentioned
- Integrations or external dependencies
- Data model or data requirements
- Performance or scalability expectations
- Security or compliance requirements
- Existing systems to integrate with
```

Replace with:

```
**Technical categories:**
- Stack or technology choices mentioned
- Integrations or external dependencies
- Data model or data requirements
- Performance or scalability expectations
- Security or compliance requirements
- Existing systems to integrate with
- UI framework / design mode — shadcn/ui, Radix, MUI, custom design system, or not mentioned (used to decide `designMode` in PROJECT_PROFILE.md)
```

**Step 3: Add a line to Step 3 classification examples table**

Find this block in Step 3:

```
| Element | Status | Reason |
|---------|--------|--------|
| "The app should be fast" | ⚠️ Ambiguous | "Fast" is undefined — no latency targets, no context |
| "Users must be authenticated" | ✅ Clear | Unambiguous requirement |
| "Success metrics" | ❌ Missing | Not mentioned anywhere |
| "Supports mobile and desktop" | ⚠️ Ambiguous | No breakpoints, no native vs. responsive specified |
```

Replace with:

```
| Element | Status | Reason |
|---------|--------|--------|
| "The app should be fast" | ⚠️ Ambiguous | "Fast" is undefined — no latency targets, no context |
| "Users must be authenticated" | ✅ Clear | Unambiguous requirement |
| "Success metrics" | ❌ Missing | Not mentioned anywhere |
| "Supports mobile and desktop" | ⚠️ Ambiguous | No breakpoints, no native vs. responsive specified |
| "Built with shadcn/ui" | ✅ Clear | UI framework set → designMode = shadcn-theme |
| No UI framework mentioned | ❌ Missing | Critical gap — designMode must be resolved before design-flow |
```

**Step 4: Verify**

Run: `Grep "designMode" skills/pm-intake/SKILL.md -n`
Expected: 2 matches (technical categories bullet + example row).

**Step 5: Commit**

```bash
git add skills/pm-intake/SKILL.md
git commit -m "pm-intake: flag UI framework / designMode as a classified element"
```

---

## Task 3: pm-handoff — read PROJECT_PROFILE and stamp designMode into briefs

**Files:**
- Modify: `skills/pm-handoff/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/pm-handoff/SKILL.md`

**Step 2: Add a new Step 1.5 between "Verify PM artifacts" and "Ask which workstreams to hand off"**

Find: `### Step 2: Ask which workstreams to hand off`

Insert immediately BEFORE that heading:

```
### Step 1.5: Read PROJECT_PROFILE.md

Read `.pm/<feature-name>/PROJECT_PROFILE.md` to get `designMode` and `uiFramework`.

If the file is missing (legacy project pre-dating this convention), prompt once:

> "Este projeto não tem PROJECT_PROFILE.md. Modo de design: **shadcn-theme** ou **custom-system**? (Escolha grava em `.pm/<feature-name>/PROJECT_PROFILE.md` — afeta design-tokens e design-components.)"

Write the file with the user's answer and continue. Do NOT re-run pm-grill.

```

**Step 3: Stamp `designMode` into the DESIGN_BRIEF.md header**

Find this block in the DESIGN_BRIEF.md template:

```
# Design Brief: [Feature/Project Name]

## Project context
```

Replace with:

```
# Design Brief: [Feature/Project Name]

**designMode:** [shadcn-theme | custom-system — from PROJECT_PROFILE.md]
**uiFramework:** [shadcn/ui | <other> | none — from PROJECT_PROFILE.md]

## Project context
```

**Step 4: Stamp `designMode` into the FRONTEND_BRIEF.md header**

Find this block in the FRONTEND_BRIEF.md template:

```
# Frontend Brief: [Feature/Project Name]

## What we're building
```

Replace with:

```
# Frontend Brief: [Feature/Project Name]

**designMode:** [shadcn-theme | custom-system — from PROJECT_PROFILE.md]
**uiFramework:** [shadcn/ui | <other> | none — from PROJECT_PROFILE.md]

## What we're building
```

**Step 5: Add a Rules entry**

Find the `## Rules` block at the bottom. Append a new bullet:

```
- Read `.pm/<feature>/PROJECT_PROFILE.md` before generating briefs. If missing, prompt once and create it — do NOT re-run pm-grill.
```

**Step 6: Verify**

Run: `Grep "PROJECT_PROFILE" skills/pm-handoff/SKILL.md -n`
Expected: at least 3 matches (Step 1.5, rules, implicit).

Run: `Grep "designMode" skills/pm-handoff/SKILL.md -n`
Expected: at least 3 matches (Step 1.5 body, DESIGN_BRIEF header, FRONTEND_BRIEF header).

**Step 7: Commit**

```bash
git add skills/pm-handoff/SKILL.md
git commit -m "pm-handoff: read PROJECT_PROFILE and stamp designMode into briefs"
```

---

## Task 4: design-flow — announce mode and handle legacy migration prompt

**Files:**
- Modify: `skills/design-flow/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/design-flow/SKILL.md`

**Step 2: Add a Step 1.5 before Step 2 (pick entry path)**

Find: `### Step 2: Pick the entry path`

Insert BEFORE that heading:

```
### Step 1.5: Read the project profile

Read `.pm/<feature>/PROJECT_PROFILE.md` to get `designMode`. If the file is missing:

> "Este projeto não tem PROJECT_PROFILE.md. Modo de design: **shadcn-theme** ou **custom-system**?"

Write the file with the answer. Then announce the mode to the user:

> "Rodando design-flow em modo **<designMode>**. `design-tokens` e `design-components` vão se ajustar a esse modo."

```

**Step 3: Append a Rules entry**

Find the `## Rules` block at the bottom. Append:

```
- Always read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1.5. Sub-skills rely on the mode being explicit.
```

**Step 4: Verify**

Run: `Grep "PROJECT_PROFILE" skills/design-flow/SKILL.md -n`
Expected: at least 2 matches.

Run: `Grep "Step 1.5" skills/design-flow/SKILL.md -n`
Expected: 1 match.

**Step 5: Commit**

```bash
git add skills/design-flow/SKILL.md
git commit -m "design-flow: read PROJECT_PROFILE and announce designMode"
```

---

## Task 5: design-tokens — add shadcn-theme mode branch

**Files:**
- Modify: `skills/design-tokens/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/design-tokens/SKILL.md`

**Step 2: Rewrite Step 1 (Load inputs) to include PROJECT_PROFILE**

Find:

```
### Step 1: Load inputs

Read:
- `.design/<feature>/DESIGN_BRIEF.md` (aesthetic direction, tone, density)
- `.design/<feature>/CODEBASE_AUDIT.md` (existing tokens to respect)
- Optionally `.design/<feature>/IA.md` (to anticipate screens that need tokens)
```

Replace with:

```
### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — get `designMode`. MUST exist (pm-handoff or design-flow creates it). Branch behavior at Step 2 based on this value.
- `.design/<feature>/DESIGN_BRIEF.md` (aesthetic direction, tone, density)
- `.design/<feature>/CODEBASE_AUDIT.md` (existing tokens to respect)
- Optionally `.design/<feature>/IA.md` (to anticipate screens that need tokens)
```

**Step 3: Add a mode branch right after Step 1 and before Step 2**

Insert immediately after the end of Step 1 section, BEFORE `### Step 2: Decide the target code format`:

```
### Step 1.5: Branch on designMode

- **`designMode: shadcn-theme`** → skip Steps 5, 6, 7 (spacing scale, typography scale, radius/motion/breakpoints/z-index). Go to the **Shadcn-theme mode** section at the end of this skill. The canonical output in that mode is a slim TOKENS.md with theme vars + radius + font only.
- **`designMode: custom-system`** → follow Steps 2–9 as written (full token system).

```

**Step 4: Append the full shadcn-theme mode section**

Append at the end of the file, AFTER the `## Rules` section:

````
---

## Shadcn-theme mode

Use when `PROJECT_PROFILE.md` has `designMode: shadcn-theme`. The designer is delivering a shadcn theme (not a full token system), so output is intentionally slim.

### Accepted inputs (any one)

- **Tweakcn export** — paste the CSS block the user exported from https://tweakcn.com (contains `--background`, `--foreground`, `--primary`, `--radius`, etc.).
- **Shadcn preset name** — one of `slate`, `zinc`, `stone`, `gray`, `neutral`, etc. Resolve to the preset's CSS block from https://ui.shadcn.com/docs/theming.
- **Manual answers** — short prompts:
  - Primary palette: one hue family (e.g., blue, violet, emerald) + saturation intensity (bold / muted)
  - Dark mode: yes/no + same hue or shifted
  - Radius: `0` | `0.3rem` | `0.5rem` | `0.75rem` | `1rem`
  - Font stack: sans + mono

Ask once at Step 1. Don't re-ask if already declared in DESIGN_BRIEF.

### Respect what already exists

If `CODEBASE_AUDIT.md` reports that the project already has shadcn CSS vars in `globals.css`, do not regenerate — only close gaps (missing dark variant, missing `--destructive`, missing `--ring`, etc.).

### Contrast verification

Run WCAG AA checks on the critical pairs:
- `--primary` on `--primary-foreground`
- `--background` on `--foreground`
- `--destructive` on `--destructive-foreground`
- `--muted-foreground` on `--background` (body text)

Flag any that fail.

### Write TOKENS.md (shadcn-theme shape)

Save to `.design/<feature>/TOKENS.md`:

```markdown
# Design Tokens: [Feature Name] (shadcn-theme mode)

## Theme source
[tweakcn export paste / shadcn preset "<name>" / manual answers]

## CSS variables

\`\`\`css
:root {
  --background: ...; --foreground: ...;
  --card: ...; --card-foreground: ...;
  --popover: ...; --popover-foreground: ...;
  --primary: ...; --primary-foreground: ...;
  --secondary: ...; --secondary-foreground: ...;
  --muted: ...; --muted-foreground: ...;
  --accent: ...; --accent-foreground: ...;
  --destructive: ...; --destructive-foreground: ...;
  --border: ...; --input: ...; --ring: ...;
  --radius: 0.5rem;
}
.dark {
  --background: ...;
  /* ... dark overrides ... */
}
\`\`\`

## Font stack
- Sans: Inter, system-ui, sans-serif
- Mono: JetBrains Mono, monospace

## What this mode does NOT include
- Full spacing scale (shadcn uses Tailwind defaults)
- Typography scale (shadcn primitives set their own)
- Motion tokens (shadcn uses Tailwind transition defaults)
- Z-index scale (shadcn components manage internally)

## Contrast verification

| Pair | Ratio | Pass |
|------|-------|------|
| `--foreground` on `--background` | 13.4:1 | ✓ AA |
| `--primary-foreground` on `--primary` | 5.8:1 | ✓ AA |
| `--muted-foreground` on `--background` | 4.7:1 | ✓ AA body |

## Applied to codebase
**Kept as-is:** [list]
**Added:** [list]
**Modified:** [list + rationale]
```

### Hand off

"Tema shadcn specced. Próximo: `design-components` em modo shadcn-theme — ele vai mapear telas para componentes shadcn em vez de re-especificar primitivos."
````

**Step 5: Append a Rules entry**

Find the `## Rules` block. Append before the Shadcn-theme mode section is appended (so Rules stays above the mode section). Add this bullet:

```
- Read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1. If `designMode: shadcn-theme`, jump to the Shadcn-theme mode section at the end — skip Steps 5–7. The output is a slim TOKENS.md, not a full token system.
```

(If sections end up in a different order from Step 4, verify the ordering: `## How to run` → `## Rules` → `## Shadcn-theme mode`.)

**Step 6: Verify**

Run: `Grep "Shadcn-theme mode" skills/design-tokens/SKILL.md -n`
Expected: at least 2 matches (Step 1.5 branch + the section heading).

Run: `Grep "tweakcn" skills/design-tokens/SKILL.md -n`
Expected: at least 2 matches (inputs bullet + shadcn preset or manual context).

Run: `Grep "PROJECT_PROFILE" skills/design-tokens/SKILL.md -n`
Expected: at least 2 matches.

**Step 7: Commit**

```bash
git add skills/design-tokens/SKILL.md
git commit -m "design-tokens: add shadcn-theme mode with slim TOKENS.md output"
```

---

## Task 6: design-components — add shadcn-theme mode with screen-centric output

**Files:**
- Modify: `skills/design-components/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/design-components/SKILL.md`

**Step 2: Rewrite Step 1 (Load inputs) to include PROJECT_PROFILE**

Find:

```
### Step 1: Load inputs

Read:
- `.design/<feature>/DESIGN_BRIEF.md` (component inventory draft)
- `.design/<feature>/IA.md` (routes and reuse map)
- `.design/<feature>/TOKENS.md` (all values components may reference)
- `.design/<feature>/CODEBASE_AUDIT.md` (existing components)
```

Replace with:

```
### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — get `designMode`. MUST exist. Branch behavior at Step 1.5.
- `.design/<feature>/DESIGN_BRIEF.md` (component inventory draft)
- `.design/<feature>/IA.md` (routes and reuse map)
- `.design/<feature>/TOKENS.md` (all values components may reference)
- `.design/<feature>/CODEBASE_AUDIT.md` (existing components)
```

**Step 3: Insert a Step 1.5 branching section**

Immediately after Step 1, BEFORE `### Step 2: Build the canonical component list`:

```
### Step 1.5: Branch on designMode

- **`designMode: custom-system`** → follow Steps 2–the rest as written. Specs every primitive and composite in full. Output file: `.design/<feature>/COMPONENT_SPECS.md`.
- **`designMode: shadcn-theme`** → follow the **Shadcn-theme mode** section at the end of this skill. Output file: `.design/<feature>/COMPONENT_SPECS.md` (same filename, different content shape — screen-centric).

```

**Step 4: Append the full shadcn-theme mode section**

Append at the end of the file, AFTER the existing `## Rules` section:

````
---

## Shadcn-theme mode

Use when `PROJECT_PROFILE.md` has `designMode: shadcn-theme`. The designer is designing screens and choosing shadcn components — NOT specifying primitives (shadcn is the contract for Button, Input, Dialog, etc.).

### What to produce

1. **Shadcn inventory** — per-feature list of shadcn components used and any customizations.
2. **Custom variants** — shadcn components that need a new variant (document the CVA diff).
3. **Roll-your-own components** — anything shadcn doesn't cover, specced as usual.
4. **Screen specs** — per screen: layout, shadcn pieces, custom components, and states (default, loading, empty, error, success).

### What to skip

- Primitive specs (Button, Input, Checkbox, etc.) — shadcn provides these.
- Token-per-component tables — implicit via shadcn CSS vars.

### Write COMPONENT_SPECS.md (shadcn-theme shape)

Save to `.design/<feature>/COMPONENT_SPECS.md`:

```markdown
# Component Specs: [Feature Name] (shadcn-theme mode)

## Mode
shadcn-theme — primitives come from the shadcn catalog. This doc covers
screens, compositions, custom variants, and roll-your-own components.

## Shadcn component inventory (used in this feature)

| Component | Source | Customization |
|-----------|--------|---------------|
| Button    | shadcn | default variants only |
| Card      | shadcn | default |
| Dialog    | shadcn | `size=lg` variant added |
| ... | ... | ... |

## Custom variants

### Button — variant `ghost-destructive`
- **Use:** secondary destructive actions (e.g., "Remove" inside a card)
- **Base:** shadcn Button `ghost`
- **Overrides:** text uses `--destructive`, hover bg `--destructive/10`
- **CVA diff:**
  \`\`\`ts
  variants: {
    variant: {
      // ...existing shadcn variants
      'ghost-destructive': 'text-destructive hover:bg-destructive/10',
    }
  }
  \`\`\`

## Roll-your-own components

### ProjectCard
- **Composition:** Card (shadcn) + Badge (shadcn) + Avatar (shadcn) + custom layout
- **Props:**
  \`\`\`ts
  interface ProjectCardProps {
    project: Project;
    onOpen: () => void;
  }
  \`\`\`
- **States:** default, hover, loading (Skeleton), empty (no-cover fallback)

## Screen specs

### Screen: Dashboard (/dashboard)
- **Layout:** SidebarLayout (roll-your-own) > PageHeader + ProjectGrid
- **Shadcn pieces:** Sidebar, Breadcrumb, Input (search), Button, Card
- **Custom:** ProjectCard, EmptyState illustration
- **States:**
  - Default: 3-column grid of ProjectCards
  - Loading: 6× Skeleton cards
  - Empty: EmptyState with "Create your first project" CTA
  - Error: Alert (shadcn, variant destructive) + retry Button

### Screen: Login (/login)
- **Layout:** CenteredCardLayout
- **Shadcn pieces:** Card, Form, Input, Button, Label, Separator
- **Custom:** none
- **States:** default, submitting (Button loading), error (FormMessage per field)
```

### Hand off

"Specs de componentes prontas em modo shadcn-theme. Próximo: `design-tasks` (slices verticais) → `frontend-flow` vai consumir este COMPONENT_SPECS e rodar `npx shadcn add <componente>` para cada item do inventário."
````

**Step 5: Append a Rules entry**

In the existing `## Rules` block, add:

```
- Read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1. If `designMode: shadcn-theme`, use the Shadcn-theme mode section at the end of this skill — screen-centric output, no primitive specs.
```

**Step 6: Verify**

Run: `Grep "shadcn-theme mode" skills/design-components/SKILL.md -n`
Expected: at least 3 matches (Step 1.5 branch + heading + mode text).

Run: `Grep "Screen specs" skills/design-components/SKILL.md -n`
Expected: 1+ match (in the shadcn-theme section template).

Run: `Grep "PROJECT_PROFILE" skills/design-components/SKILL.md -n`
Expected: at least 2 matches.

**Step 7: Commit**

```bash
git add skills/design-components/SKILL.md
git commit -m "design-components: add shadcn-theme mode with screen-centric COMPONENT_SPECS"
```

---

## Task 7: frontend-stack — add PROJECT_PROFILE conflict check

**Files:**
- Modify: `skills/frontend-stack/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/frontend-stack/SKILL.md`

**Step 2: Rewrite Step 1 (Load intake) to include PROJECT_PROFILE**

Find:

```
### Step 1: Load intake

Read `.frontend/<feature>/FRONTEND_INTAKE.md`. If missing, direct user to run `frontend-brief-intake` first.

Also read `.design/<feature>/TOKENS.md` if present — it constrains styling choices.
```

Replace with:

```
### Step 1: Load intake

Read `.frontend/<feature>/FRONTEND_INTAKE.md`. If missing, direct user to run `frontend-brief-intake` first.

Read `.pm/<feature>/PROJECT_PROFILE.md` to get `designMode` and `uiFramework`. If missing, prompt the user once and create it (same migration prompt used by `pm-handoff` and `design-flow`).

Also read `.design/<feature>/TOKENS.md` if present — it constrains styling choices.
```

**Step 3: Add a conflict check after Step 2 (decision table)**

Find the end of Step 2 (the decision table ends with `| Hosting | ... | Vercel |`). Immediately after that and before Step 3, insert:

```
### Step 2.5: Profile conflict check

Cross-check the Component library row against `PROJECT_PROFILE.md`:

- If `designMode: shadcn-theme` AND Component library ≠ `shadcn/ui` (customized) → **abort**:

  > "Conflito: PROJECT_PROFILE diz `shadcn-theme` mas você escolheu `[X]` como Component library. Reconcile antes de continuar: ou troque o Component library para `shadcn/ui`, ou edite `.pm/<feature>/PROJECT_PROFILE.md` para `designMode: custom-system`."

- If `designMode: custom-system` AND Component library = `shadcn/ui` → proceed, but note in the decision log that shadcn is being used as a primitive base under a custom token system (valid, just explicit).

```

**Step 4: Verify**

Run: `Grep "PROJECT_PROFILE" skills/frontend-stack/SKILL.md -n`
Expected: at least 2 matches.

Run: `Grep "Conflito" skills/frontend-stack/SKILL.md -n`
Expected: 1 match.

**Step 5: Commit**

```bash
git add skills/frontend-stack/SKILL.md
git commit -m "frontend-stack: add PROJECT_PROFILE conflict check"
```

---

## Task 8: frontend-components — consume shadcn-mode COMPONENT_SPECS

**Files:**
- Modify: `skills/frontend-components/SKILL.md`

**Step 1: Read the current skill file**

Run: `Read skills/frontend-components/SKILL.md`

**Step 2: Rewrite Step 1 (Load inputs) to include PROJECT_PROFILE**

Find:

```
### Step 1: Load inputs

Read:
- `.design/<feature>/COMPONENT_SPECS.md` (visual contract per component — MUST exist)
- `.design/<feature>/TOKENS.md` (valid token names to reference)
- `.frontend/<feature>/FRONTEND_STACK.md` (framework + styling idioms)

If `COMPONENT_SPECS.md` is missing, abort. If `TOKENS.md` is missing, warn — you'll proceed referencing tokens by name as declared in SPECS, but downstream implementation may hit missing variables.
```

Replace with:

```
### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — get `designMode`. MUST exist.
- `.design/<feature>/COMPONENT_SPECS.md` (visual contract per component — MUST exist). Content shape depends on `designMode`.
- `.design/<feature>/TOKENS.md` (valid token names to reference)
- `.frontend/<feature>/FRONTEND_STACK.md` (framework + styling idioms)

If `COMPONENT_SPECS.md` is missing, abort. If `TOKENS.md` is missing, warn — you'll proceed referencing tokens by name as declared in SPECS, but downstream implementation may hit missing variables.
```

**Step 3: Add a Step 1.5 branching section**

Immediately after Step 1, BEFORE `### Step 2: Enumerate components to plan`:

```
### Step 1.5: Branch on designMode

- **`designMode: custom-system`** → follow Steps 2–6 as written. Plan every primitive, composite, and feature component.
- **`designMode: shadcn-theme`** → see the **Shadcn-theme mode** section at the end of this skill. In that mode COMPONENT_SPECS is screen-centric: plan the custom variants and roll-your-own components, and emit a `shadcn add` list for primitives instead of re-planning them.

```

**Step 4: Append the shadcn-theme mode section**

Append at the end of the file, AFTER the existing `## Rules` section:

````
---

## Shadcn-theme mode

Use when `PROJECT_PROFILE.md` has `designMode: shadcn-theme`. COMPONENT_SPECS lists shadcn inventory + custom variants + roll-your-own + screens.

### What to produce in COMPONENT_PLAN.md

1. **Shadcn install plan** — one `npx shadcn add <component>` per row of the shadcn inventory. No per-primitive spec (shadcn ships the contract).
2. **Custom variants** — for each custom variant, plan the CVA extension file path (e.g., `components/ui/button.tsx` already exists — we extend its `variants.variant` map). Include the exact CVA diff from COMPONENT_SPECS.
3. **Roll-your-own components** — planned exactly like current Step 3 (target file path, props, state, data deps, token usage, a11y, responsive, test plan).
4. **Screen implementation order** — topological order of screens from COMPONENT_SPECS, each listing what shadcn needs to be installed first and what custom/roll-your-own components it depends on.

### What to skip

- Per-primitive plans for shadcn-provided components (Button, Input, Card, Dialog, etc.).
- Token maps per primitive — implicit via shadcn CSS vars.

### Write COMPONENT_PLAN.md (shadcn-theme shape)

Save to `.frontend/<feature>/COMPONENT_PLAN.md`:

```markdown
# Component Plan: [Feature Name] (shadcn-theme mode)

## Shadcn install plan

\`\`\`bash
npx shadcn add button
npx shadcn add card
npx shadcn add dialog
npx shadcn add form
npx shadcn add input
# ... one per row of inventory in COMPONENT_SPECS
\`\`\`

## Custom variants

### Button — `ghost-destructive`
- **File:** `components/ui/button.tsx` (edit after `npx shadcn add button`)
- **Change:** add variant entry to the `buttonVariants` CVA config
- **CVA diff:**
  \`\`\`ts
  variants: {
    variant: {
      // ...existing variants from shadcn default
      'ghost-destructive': 'text-destructive hover:bg-destructive/10',
    }
  }
  \`\`\`
- **Test:** unit (RTL) — render Button with variant + assert classname includes `text-destructive`.

## Roll-your-own components

### ProjectCard
- **File:** `components/feature/project-card.tsx`
- **Exported:** `ProjectCard` (named)
- **Props:**
  \`\`\`ts
  interface ProjectCardProps {
    project: Project;
    onOpen: () => void;
  }
  \`\`\`
- **Internal state:** none (presentational)
- **Data deps:** none (parent passes `project`)
- **Token usage:** via shadcn vars only — `bg-card`, `text-card-foreground`, `border`, `hover:bg-accent`
- **A11y:** root `<article>` + heading `<h3>` for project name; whole card focusable via keyboard (tabindex 0, onKeyDown for Enter/Space)
- **Responsive:** stacks vertically < `md`; side-by-side >= `md`
- **Test:** unit (RTL) — render + click + keyboard activation; integration — inside ProjectGrid

## Screen implementation order

1. Install shadcn Button, Card, Input — needed everywhere
2. Implement roll-your-own ProjectCard — depends on shadcn Card, Badge, Avatar
3. Implement SidebarLayout — depends on shadcn Sidebar, Breadcrumb
4. Screen: Dashboard — depends on SidebarLayout + ProjectCard
5. Install shadcn Form, Label, Separator
6. Implement CenteredCardLayout
7. Screen: Login — depends on CenteredCardLayout + shadcn Form
```

### Hand off

"Plan de componentes em modo shadcn-theme pronto. `frontend-tasks` vai transformar em slices verticais — primeira slice normalmente é 'scaffold + install shadcn base'."
````

**Step 5: Append a Rules entry**

In the existing `## Rules` block, add:

```
- Read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1. If `designMode: shadcn-theme`, use the Shadcn-theme mode section — emit `npx shadcn add` commands instead of per-primitive plans.
```

**Step 6: Verify**

Run: `Grep "Shadcn-theme mode" skills/frontend-components/SKILL.md -n`
Expected: at least 3 matches.

Run: `Grep "npx shadcn add" skills/frontend-components/SKILL.md -n`
Expected: at least 2 matches (in the mode text + template).

Run: `Grep "PROJECT_PROFILE" skills/frontend-components/SKILL.md -n`
Expected: at least 2 matches.

**Step 7: Commit**

```bash
git add skills/frontend-components/SKILL.md
git commit -m "frontend-components: consume shadcn-mode COMPONENT_SPECS and emit shadcn add plan"
```

---

## Task 9: End-to-end read-through verification

**Files:** (no edits — read-only verification)

**Step 1: Walk the full flow mentally using the edited skills**

Read in this order:
1. `skills/pm-grill/SKILL.md` — look at Mode B § Step B2 Technical branches. The Delivery profile branch must mention `shadcn-theme` and `custom-system`.
2. `skills/pm-intake/SKILL.md` — look at Step 2 Technical categories. A bullet must exist for UI framework / designMode.
3. `skills/pm-handoff/SKILL.md` — look for Step 1.5. The DESIGN_BRIEF and FRONTEND_BRIEF templates must have `**designMode:**` in their headers.
4. `skills/design-flow/SKILL.md` — look for Step 1.5 reading PROJECT_PROFILE.
5. `skills/design-tokens/SKILL.md` — look for Step 1.5 branching + the Shadcn-theme mode section at the bottom.
6. `skills/design-components/SKILL.md` — same pattern.
7. `skills/frontend-stack/SKILL.md` — look for Step 2.5 conflict check.
8. `skills/frontend-components/SKILL.md` — look for Step 1.5 branching + Shadcn-theme mode section.

**Step 2: Run cross-file consistency greps**

```bash
# Every skill that should reference PROJECT_PROFILE
Grep "PROJECT_PROFILE" skills/pm-grill/SKILL.md skills/pm-handoff/SKILL.md skills/design-flow/SKILL.md skills/design-tokens/SKILL.md skills/design-components/SKILL.md skills/frontend-stack/SKILL.md skills/frontend-components/SKILL.md -l
```

Expected: all 7 files match.

```bash
# designMode values used consistently
Grep "shadcn-theme" skills/ -r -l
```

Expected: matches across pm-grill, pm-intake, pm-handoff, design-flow, design-tokens, design-components, frontend-stack, frontend-components. (8 files.)

**Step 3: Spot-check narrative coherence**

Open `skills/design-flow/SKILL.md` and `skills/design-tokens/SKILL.md` side by side. Confirm:
- design-flow announces "modo shadcn-theme" at Step 1.5.
- design-tokens' Step 1.5 branch sends the user to the Shadcn-theme mode section.
- The migration prompt wording is identical in design-flow and pm-handoff (copy-paste check).

If any of the above fails, open a task to fix the drift and re-run verification.

**Step 4: No commit** (this task is read-only).

---

## Out of scope for this plan (follow-ups)

- README.md "Project modes" section — low priority; design doc explicitly deferred.
- Automating tweakcn import via URL fetch — v1 accepts paste only.
- Additional `designMode` values (`material-ui`, `chakra`, etc.) — add when a real project demands.
- pm-review and pm-architecture awareness — currently not critical, they don't shape TOKENS/COMPONENT_SPECS content. Revisit if friction appears.

---

## Execution Handoff

Plan complete and saved to `docs/plans/2026-04-18-shadcn-mvp-mode.md`. Two execution options:

1. **Subagent-Driven (this session)** — dispatch fresh subagent per task, review between tasks, fast iteration.
2. **Parallel Session (separate)** — open new session with executing-plans, batch execution with checkpoints.

Which approach?
