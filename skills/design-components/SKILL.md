---
name: design-components
description: Reads `.design/<feature>/DESIGN_BRIEF.md`, `IA.md`, and `TOKENS.md` and produces `.design/<feature>/COMPONENT_SPECS.md` — per-component spec (status, anatomy, props API, variants, states, a11y, responsive behavior, dependencies). Classifies each as reuse / modify / new. Does not write implementation code. Use when design-flow invokes it or when user says "spec the components", "design the components", "component inventory". Not for generating JSX/TSX implementations (that's frontend-flow's job) or for generic component-library discussions.
---

# Design Components — Component Specs

Produces the canonical contract for every UI component, consumed by `frontend-components` downstream.

## How to run

### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — get `designMode`. MUST exist. Branch behavior at Step 1.5.
- `.design/<feature>/DESIGN_BRIEF.md` (component inventory draft)
- `.design/<feature>/IA.md` (routes and reuse map)
- `.design/<feature>/TOKENS.md` (all values components may reference)
- `.design/<feature>/CODEBASE_AUDIT.md` (existing components)

### Step 1.5: Branch on designMode

- **`designMode: custom-system`** → follow Steps 2–the rest as written. Specs every primitive and composite in full. Output file: `.design/<feature>/COMPONENT_SPECS.md`.
- **`designMode: shadcn-theme`** → follow the **Shadcn-theme mode** section at the end of this skill. Output file: `.design/<feature>/COMPONENT_SPECS.md` (same filename, different content shape — screen-centric).

### Step 2: Build the canonical component list

Cross-reference:
- Component inventory from the brief
- Routes × screens from IA (each screen implies specific components)
- Existing components from the codebase audit

Deduplicate. Group by category:
- **Primitives** (Button, Input, Select, Checkbox, Radio, Switch, Badge, Avatar)
- **Composites** (Card, Modal, Drawer, Popover, Toast, Tabs, Accordion)
- **Feature components** (ProjectCard, TaskList, UserMenu — feature-specific)
- **Layout** (Header, Footer, Sidebar, PageShell, EmptyState)

### Step 3: Classify each component

For each component, set status:
- **reuse** — exists in codebase, use as-is
- **modify** — exists but needs a new variant / prop
- **new** — does not exist, spec from scratch

Skip specs for `reuse` components. Full spec for `modify` and `new`.

### Step 4: Spec each component (modify / new)

For each component requiring a spec, write:

**Anatomy** — named slots (header, body, footer, trigger, ...)

**Props API** — TypeScript-style signature. Be strict about required vs optional:

```ts
interface ButtonProps {
  variant?: 'primary' | 'secondary' | 'ghost' | 'destructive'; // default: 'primary'
  size?: 'sm' | 'md' | 'lg';                                    // default: 'md'
  loading?: boolean;
  leftIcon?: ReactNode;
  rightIcon?: ReactNode;
  disabled?: boolean;
  children: ReactNode;
  onClick?: (e: MouseEvent) => void;
}
```

**Variants** — enumerate each variant with its intended use:
- `primary` — main CTA, one per screen section
- `secondary` — alternative action, same weight
- `ghost` — tertiary / destructive-adjacent actions
- `destructive` — delete / permanent actions

**States** — cover ALL eight where applicable:
- default, hover, focus, active (pressed), disabled, loading, error, empty (for data components)

Describe each state in terms of token changes (`bg-accent` → `bg-accent-hover`), not raw colors.

**Accessibility contract**:
- ARIA role (if not native)
- Keyboard interaction (Tab, Enter, Space, Escape, Arrow keys)
- Screen reader label expectations
- Focus management (e.g., modals trap focus, drawers return focus to trigger)
- Contrast requirements for each state

**Responsive behavior**:
- Mobile baseline (375px): how does it render?
- Tablet (768px): changes?
- Desktop (1024+px): changes?
- Does it reflow, stack, hide elements, switch to a sheet/drawer?

**Dependencies** — other components it composes (e.g., `Modal` uses `Button`).

### Step 5: Order by dependency

Topologically sort — primitives first, then composites that use them, then feature components. `frontend-components` will implement in this order.

### Step 6: Write COMPONENT_SPECS.md

Save to `.design/<feature>/COMPONENT_SPECS.md`:

```markdown
# Component Specs: [Feature Name]

## Inventory

| Component | Category | Status | Used by |
|-----------|----------|--------|---------|
| Button | primitive | reuse | (shadcn) — used by 14 screens |
| Modal | composite | modify | needs `destructive` variant |
| ProjectCard | feature | new | /dashboard, /projects |
| EmptyState | layout | new | everywhere |

## Dependency order (build this order)

1. Button (primitive, reuse — no spec needed)
2. Modal (composite — spec below)
3. EmptyState (layout, new — spec below)
4. ProjectCard (feature, new — uses Button, Avatar — spec below)

---

## Modal

**Status:** modify (existing component lacks `destructive` variant)

**Anatomy:**
- `trigger` — slot that opens the modal (any clickable)
- `header` — title + optional close button
- `body` — scrollable content area
- `footer` — action buttons, right-aligned on desktop, stacked on mobile

**Props:**
```ts
interface ModalProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  variant?: 'default' | 'destructive';  // NEW
  size?: 'sm' | 'md' | 'lg' | 'fullscreen';
  trigger?: ReactNode;
  title: string;
  description?: string;
  children: ReactNode;
  footer?: ReactNode;
}
```

**Variants:**
- `default` — neutral framing (white header strip)
- `destructive` — red accent on header, for delete confirmations. Primary button in footer is `variant='destructive'`.

**States:**
- closed: no DOM presence (unmounted)
- opening: backdrop fades in 150ms, content slides up 250ms
- open: focus trapped, body scroll locked
- closing: reverse of opening
- error (form inside modal): inline banner above footer, does not close modal

**Accessibility:**
- role: `dialog`, `aria-modal: true`
- `aria-labelledby` → title id
- `aria-describedby` → description id (if present)
- Escape closes modal; trap focus inside; return focus to trigger on close
- Backdrop click closes (overrideable)

**Responsive:**
- Mobile (< 768): bottom sheet, sticks to bottom, max-height 90vh
- Tablet+: centered dialog, max-width depending on size

**Dependencies:** uses `Button` (footer actions).

---

## EmptyState

**Status:** new

[...full spec as above...]

---

## ProjectCard

**Status:** new
[...]
```

### Step 7: Hand off

Tell the user: "Specs recorded for [N modify + M new] components. Next: `design-tasks` orders these into vertical slices for the frontend build."

## Rules

- Props API uses TypeScript syntax — even if the project is Vue/Svelte, this is the contract language.
- Every variant must have a named use case — no unused variants.
- Every component must declare its mobile behavior — default is "reflows normally" but say so.
- Reference tokens by name (`bg-accent-hover`), never raw hex/rgb values.
- Mark status explicitly — `frontend-components` skips components with status `reuse`.
- Don't include implementation code. The contract is enough; `frontend-components` does the code mapping.
- Read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1. If `designMode: shadcn-theme`, use the Shadcn-theme mode section at the end of this skill — screen-centric output, no primitive specs.

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

````markdown
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
  ```ts
  variants: {
    variant: {
      // ...existing shadcn variants
      'ghost-destructive': 'text-destructive hover:bg-destructive/10',
    }
  }
  ```

## Roll-your-own components

### ProjectCard
- **Composition:** Card (shadcn) + Badge (shadcn) + Avatar (shadcn) + custom layout
- **Props:**
  ```ts
  interface ProjectCardProps {
    project: Project;
    onOpen: () => void;
  }
  ```
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
````

### Hand off

"Specs de componentes prontas em modo shadcn-theme. Próximo: `design-tasks` (slices verticais) → `frontend-flow` vai consumir este COMPONENT_SPECS e rodar `npx shadcn add <componente>` para cada item do inventário."
