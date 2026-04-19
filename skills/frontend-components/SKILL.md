---
name: frontend-components
description: Reads `.design/<feature>/COMPONENT_SPECS.md` and `TOKENS.md` plus `.frontend/<feature>/FRONTEND_STACK.md`, and produces `.frontend/<feature>/COMPONENT_PLAN.md` — per-component implementation contract (file path, exported name, props interface, variants, internal state, data deps, a11y implementation, test plan). Planning only — does not write code. Use when frontend-flow invokes it or when user says "plan the components", "map specs to code". Not for writing actual JSX/TSX (that happens in the implementation step driven by frontend-tasks) or for ad-hoc component design.
---

# Frontend Components — Code Contract Per Component

Maps every design spec to a concrete implementation plan. Does NOT write code.

## How to run

### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — get `designMode`. MUST exist.
- `.design/<feature>/COMPONENT_SPECS.md` (visual contract per component — MUST exist). Content shape depends on `designMode`.
- `.design/<feature>/TOKENS.md` (valid token names to reference)
- `.frontend/<feature>/FRONTEND_STACK.md` (framework + styling idioms)

If `COMPONENT_SPECS.md` is missing, abort. If `TOKENS.md` is missing, warn — you'll proceed referencing tokens by name as declared in SPECS, but downstream implementation may hit missing variables.

### Step 1.5: Branch on designMode

- **`designMode: custom-system`** → follow Steps 2–6 as written. Plan every primitive, composite, and feature component.
- **`designMode: shadcn-theme`** → see the **Shadcn-theme mode** section at the end of this skill. In that mode COMPONENT_SPECS is screen-centric: plan the custom variants and roll-your-own components, and emit a `shadcn add` list for primitives instead of re-planning them.

### Step 2: Enumerate components to plan

Skip entries in SPECS with status `reuse` — those don't need planning.

For each `modify` or `new` component, classify:
- **Primitive** (Button, Input) → `components/ui/`
- **Composite** (Modal, Tabs) → `components/ui/` or `components/` based on project convention
- **Feature** (ProjectCard, TaskList) → `components/feature/` or colocate with route
- **Layout** (PageShell, EmptyState) → `components/layout/`

### Step 3: Plan each component

Per component, write:

**Target file path** — `components/ui/modal.tsx` (follow FRONTEND_STACK conventions)

**Exported symbols** — default vs named; compound components (e.g., `Modal.Trigger`, `Modal.Content`)

**Props interface** — TypeScript exactly as declared in SPECS, plus any implementation-only props (`className`, `ref`). If the SPEC lists variants, wire them to `class-variance-authority` (cva) slots.

**Internal state shape** — what this component manages locally (controlled vs uncontrolled patterns explicit)

**Data dependencies**:
- Server-fetched via props (parent fetches, passes down)
- Server-fetched via hook (TanStack Query within component)
- Client state (Zustand store reference)
- None (presentational)

**Token usage map** — which design tokens from TOKENS.md this component consumes for each state (no raw hex allowed)

**Accessibility implementation**:
- What primitive from Radix / shadcn to build on (if applicable)
- Which ARIA attributes wired where
- Keyboard shortcuts implemented
- Focus management rules

**Responsive implementation**:
- Breakpoint-specific layouts (Tailwind classes or framework primitives)
- Components that swap entirely at mobile (e.g., Modal → Sheet)

**Test plan**:
- Unit: render + props variants (RTL)
- Behavior: interaction paths (click, key events)
- E2E: only if the component is part of a critical journey

### Step 4: Order topologically

Primitives first (Button, Input), then composites (Modal uses Button), then features (ProjectCard uses Button + Avatar). Match the order FRONTEND_TASKS will implement.

### Step 5: Flag blockers

- Missing SPEC for a component mentioned in IA routes — flag.
- Missing token for a state the SPEC demands — flag and suggest the token name needed.
- Variant names conflict with shadcn defaults (e.g., shadcn Button has `destructive`; SPEC adds `ghost-destructive`) — call out the naming strategy.

### Step 6: Write COMPONENT_PLAN.md

Save to `.frontend/<feature>/COMPONENT_PLAN.md`:

````markdown
# Component Plan: [Feature Name]

**Design specs:** .design/feature-x/COMPONENT_SPECS.md
**Tokens:** .design/feature-x/TOKENS.md
**Stack:** FRONTEND_STACK.md

## Implementation order (topological)

1. Button (primitive, reuse — no plan needed)
2. Modal (composite, modify)
3. EmptyState (layout, new)
4. ProjectCard (feature, new — uses Button, Avatar)

---

## Modal

**Target file:** `components/ui/modal.tsx`
**Exports:** `Modal` (compound: `Modal.Trigger`, `Modal.Content`, `Modal.Header`, `Modal.Body`, `Modal.Footer`)
**Builds on:** `@radix-ui/react-dialog`

**Props:**
```ts
import { cva, type VariantProps } from 'class-variance-authority';

const modalContentVariants = cva(
  'fixed z-modal bg-surface text-text rounded-lg shadow-xl',
  {
    variants: {
      variant: {
        default: 'border border-border',
        destructive: 'border border-danger',
      },
      size: {
        sm: 'max-w-sm', md: 'max-w-md', lg: 'max-w-lg',
        fullscreen: 'inset-0 max-w-none rounded-none',
      },
    },
    defaultVariants: { variant: 'default', size: 'md' },
  },
);

interface ModalProps extends VariantProps<typeof modalContentVariants> {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  trigger?: ReactNode;
  title: string;
  description?: string;
  children: ReactNode;
  footer?: ReactNode;
}
```

**Internal state:** none (fully controlled via `open`/`onOpenChange`)

**Data dependencies:** none

**Token usage:**
- `bg-surface`, `text-text`, `border` for default variant
- `border-danger` for destructive variant
- `z-modal` token for stacking
- `bg-overlay` for backdrop

**Accessibility:**
- Wraps `Dialog.Root` / `Dialog.Content` / `Dialog.Title` / `Dialog.Description` from Radix
- Focus trap built-in via Radix
- Escape key closes (Radix default)
- `aria-labelledby` → title id; `aria-describedby` → description id (Radix handles)

**Responsive:**
- Mobile (`< md`): swap to bottom sheet — class `md:fixed md:inset-0 md:items-center` + `items-end` on backdrop flex container
- Tablet+: centered dialog

**Test plan:**
- Unit: renders with props, fires onOpenChange on backdrop click
- Behavior: Escape closes, focus returns to trigger on close
- E2E: skip (covered by consuming feature tests)

---

## EmptyState

**Target file:** `components/layout/empty-state.tsx`
**Exports:** `EmptyState` (default)
**Builds on:** plain HTML + cva for variants

[...similar plan structure...]

---

## ProjectCard

**Target file:** `components/feature/project-card.tsx`
**Dependencies:** `Button`, `Avatar`

[...similar...]

## Blockers / open questions

- COMPONENT_SPECS.md references a `Toast` component but no spec exists yet — need design to fill in before implementation
- Variant `accent-subtle` referenced but missing from TOKENS.md — need design to add
````

### Step 7: Hand off

Tell the user: "Component plans recorded for [N] components ([M] primitives, [K] composites, [L] features). Next: `frontend-tasks` orders these into vertical slices with design-task dependencies."

## Rules

- Planning only — do NOT write `.tsx` files from this skill.
- Props API matches SPEC exactly; if you need extra props (className, ref), add them but note "implementation" vs "from spec".
- Never hardcode colors/spacing — every visual rule references a token by name.
- When building on Radix / shadcn, declare which primitives you wrap. This is the contract for `frontend-tasks` and for `design-review` to verify.
- Missing specs or missing tokens are blockers — surface them, don't invent fills.
- Read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1. If `designMode: shadcn-theme`, use the Shadcn-theme mode section — emit `npx shadcn add` commands instead of per-primitive plans.

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

`````markdown
# Component Plan: [Feature Name] (shadcn-theme mode)

## Shadcn install plan

```bash
npx shadcn add button
npx shadcn add card
npx shadcn add dialog
npx shadcn add form
npx shadcn add input
# ... one per row of inventory in COMPONENT_SPECS
```

## Custom variants

### Button — `ghost-destructive`
- **File:** `components/ui/button.tsx` (edit after `npx shadcn add button`)
- **Change:** add variant entry to the `buttonVariants` CVA config
- **CVA diff:**
  ```ts
  variants: {
    variant: {
      // ...existing variants from shadcn default
      'ghost-destructive': 'text-destructive hover:bg-destructive/10',
    }
  }
  ```
- **Test:** unit (RTL) — render Button with variant + assert classname includes `text-destructive`.

## Roll-your-own components

### ProjectCard
- **File:** `components/feature/project-card.tsx`
- **Exported:** `ProjectCard` (named)
- **Props:**
  ```ts
  interface ProjectCardProps {
    project: Project;
    onOpen: () => void;
  }
  ```
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
`````

### Hand off

"Plan de componentes em modo shadcn-theme pronto. `frontend-tasks` vai transformar em slices verticais — primeira slice normalmente é 'scaffold + install shadcn base'."
