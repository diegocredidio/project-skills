---
name: design-tokens
description: Reads `.design/<feature>/DESIGN_BRIEF.md` and `CODEBASE_AUDIT.md` and produces `.design/<feature>/TOKENS.md` — the canonical visual token system (Brand & Style preamble, color scales, spacing, typography, radius, motion, elevation & depth, breakpoints, plus Do's and Don'ts guardrails). Dark-first by default, light as variation; elevation supports shadow-based, tonal-layers, or glass-blur strategies. Emits spec in markdown plus fenced code snippets in the project's format (CSS vars / Tailwind config / theme.ts) — does not write code into the user's source tree. Use when design-flow invokes it or when user says "design tokens", "generate the token system", "build the color palette". Not for component-level styling decisions (that's design-components) or general color-theory consulting.
---

# Design Tokens — Visual System Specification

Generates the semantic token layer that every downstream UI decision will reference.

## How to run

### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — get `designMode`. MUST exist (pm-handoff or design-flow creates it). Branch behavior at Step 1.5 based on this value.
- `.design/<feature>/DESIGN_BRIEF.md` (aesthetic direction, tone, density)
- `.design/<feature>/CODEBASE_AUDIT.md` (existing tokens to respect)
- Optionally `.design/<feature>/IA.md` (to anticipate screens that need tokens)

### Step 1.5: Branch on designMode

- **`designMode: shadcn-theme`** → skip Steps 5, 6, 7 (spacing scale, typography scale, radius/motion/elevation/breakpoints/z-index). Go to the **Shadcn-theme mode** section at the end of this skill. The canonical output in that mode is a slim TOKENS.md with theme vars + radius + font + minimal elevation + do's and don'ts.
- **`designMode: custom-system`** → follow Steps 2–9 as written (full token system).

### Step 2: Decide the target code format

Match the project's existing style system:
- **Tailwind v3+** → `tailwind.config.ts` `theme.extend` + CSS variables in `globals.css`
- **shadcn/ui** → HSL CSS variables in `globals.css` under `:root` and `.dark`
- **CSS-in-JS** (stitches, emotion) → `theme.ts`
- **Plain CSS** → CSS variables on `:root` / `[data-theme]`

Output snippets in all relevant formats, but mark the primary one clearly.

### Step 3: Respect what already exists

If the codebase has a functional token system, **do not regenerate it**. Only close gaps:
- Missing scales
- Missing states (hover, focus, active not differentiated)
- Missing dark-mode mapping
- Missing motion tokens

Document what was kept vs. what was added.

### Step 4: Build the color system (dark-first)

Use semantic names, not raw color names. Every token must have a `dark` and `light` value.

Required token families:

- **Background layers:** `bg`, `bg-subtle`, `bg-muted`, `bg-elevated`, `bg-overlay`
- **Surface:** `surface`, `surface-hover`, `surface-pressed`
- **Text:** `text`, `text-muted`, `text-subtle`, `text-inverse`
- **Border:** `border`, `border-strong`, `border-subtle`
- **Accent (brand):** `accent`, `accent-hover`, `accent-pressed`, `accent-subtle`, `accent-foreground`
- **Status:** `success`, `warning`, `danger`, `info` — each with `-subtle`, `-foreground`
- **Focus ring:** `focus-ring` (usually accent with offset)

Dark-first rule: design the dark palette intentionally, then derive light as a variation — not as a simple inversion.

### Step 5: Build the spacing scale (4px base)

Scale: `0, 1 (4), 2 (8), 3 (12), 4 (16), 5 (20), 6 (24), 8 (32), 10 (40), 12 (48), 16 (64), 20 (80), 24 (96)`.

### Step 6: Build the typography scale

Use `clamp()` for mobile-first fluid sizes:

| Token | Size (clamp) | Line-height | Weight | Use |
|-------|--------------|-------------|--------|-----|
| `text-xs`  | 0.75rem fixed | 1.5 | 500 | badges, timestamps |
| `text-sm`  | 0.875rem fixed | 1.5 | 500 | secondary UI text |
| `text-base` | 1rem fixed | 1.5 | 400 | body |
| `text-lg`  | `clamp(1.125rem, ...)` | 1.4 | 500 | emphasized body |
| `text-xl`  | `clamp(1.25rem, ...)` | 1.3 | 600 | small headings |
| `text-2xl` | `clamp(1.5rem, ...)` | 1.2 | 600 | headings |
| `text-3xl` | `clamp(1.875rem, ...)` | 1.15 | 700 | section headings |
| `text-4xl` | `clamp(2.25rem, ...)` | 1.1 | 700 | page titles |
| `text-5xl` | `clamp(3rem, ...)` | 1.05 | 700 | hero |

Declare the font stack (primary, monospace, fallback) based on the brief.

### Step 7: Build radius, motion, elevation, breakpoints

- **Radius:** `none (0), sm (4), md (8), lg (12), xl (16), 2xl (24), full (9999)`
- **Motion durations:** `fast (100ms), base (150ms), slow (250ms), slower (400ms)`
- **Motion easing:** `ease-out (default), ease-in-out (layout), spring` — define `cubic-bezier` values
- **Breakpoints:** `sm 375, md 768, lg 1024, xl 1280, 2xl 1440` (mobile-first min-width queries)
- **Z-index:** `base (0), sticky (10), dropdown (20), overlay (30), modal (40), popover (50), toast (60)`
- **Elevation & depth:** decide the hierarchy strategy based on the brief — pick ONE primary approach and describe fallbacks if used together. Output a scale with at least 4 levels.

  **Strategies (pick one primary):**
  - **Shadow-based** (default for most UIs) — stacked shadows with growing y-offset, spread, and blur. Every level has a dark and light value (shadows are less visible in dark mode and usually need higher opacity).
  - **Tonal layers** (flat / material-you style) — surface color shifts (`surface`, `surface-subtle`, `surface-elevated`, `surface-overlay` from Step 4) convey hierarchy without shadows. Document the mapping explicitly.
  - **Glass / backdrop-blur** (glassmorphism) — `backdrop-filter: blur(Npx)` + translucent background + 1px light border. Define blur levels and border alphas.

  **Required tokens per level** (regardless of strategy): `elevation-0` (flat/base), `elevation-1` (card resting), `elevation-2` (raised card / dropdown), `elevation-3` (modal / popover), `elevation-4` (toast / top-most).

  **Shadow scale defaults** (when strategy = shadow-based):

  | Level | Dark value | Light value |
  |-------|------------|-------------|
  | `elevation-0` | `none` | `none` |
  | `elevation-1` | `0 1px 2px 0 rgb(0 0 0 / 0.6)` | `0 1px 2px 0 rgb(0 0 0 / 0.05)` |
  | `elevation-2` | `0 4px 8px -2px rgb(0 0 0 / 0.6), 0 2px 4px -2px rgb(0 0 0 / 0.4)` | `0 4px 8px -2px rgb(0 0 0 / 0.1), 0 2px 4px -2px rgb(0 0 0 / 0.06)` |
  | `elevation-3` | `0 12px 24px -6px rgb(0 0 0 / 0.7), 0 8px 16px -4px rgb(0 0 0 / 0.5)` | `0 12px 24px -6px rgb(0 0 0 / 0.12), 0 8px 16px -4px rgb(0 0 0 / 0.08)` |
  | `elevation-4` | `0 24px 48px -12px rgb(0 0 0 / 0.8)` | `0 24px 48px -12px rgb(0 0 0 / 0.18)` |

### Step 8: Write TOKENS.md

Save to `.design/<feature>/TOKENS.md`:

```markdown
# Design Tokens: [Feature Name]

**Target format:** [Tailwind + shadcn CSS vars / theme.ts / plain CSS]
**Dark mode:** dark-first, light as variation

## Brand & Style

[2–4 sentences describing the aesthetic intent behind these tokens — brand personality, emotional response, density, reference vibe. Pulled from DESIGN_BRIEF.md's "Aesthetic direction" section, condensed. This preamble tells downstream agents WHY the palette leans dark / muted / vibrant / glass, so they can judge edge cases the tokens don't cover explicitly.]

## Color system

### Semantic tokens

| Token | Dark value | Light value | Contrast (AA text on top) |
|-------|------------|-------------|--------------------------|
| `bg` | `hsl(222 15% 7%)` | `hsl(0 0% 100%)` | ✓ |
| `surface` | `hsl(222 15% 11%)` | `hsl(220 15% 98%)` | ✓ |
| `text` | `hsl(0 0% 96%)` | `hsl(222 47% 11%)` | ✓ |
| ... | | | |

### Status palette

| Token | Dark | Light |
|-------|------|-------|
| `success`  | ... | ... |
| `warning`  | ... | ... |
| `danger`   | ... | ... |
| `info`     | ... | ... |

## Spacing

`0, 4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 80, 96` (px).

## Typography

[Font stack]
[Scale table]

## Radius / motion / breakpoints / z-index

[Tables above]

## Elevation & depth

**Strategy:** [shadow-based / tonal-layers / glass-blur] — [one sentence on why, tied to the Brand & Style preamble]

| Token | Role | Dark value | Light value |
|-------|------|------------|-------------|
| `elevation-0` | flat / base surface | ... | ... |
| `elevation-1` | resting card, list row | ... | ... |
| `elevation-2` | raised card, dropdown | ... | ... |
| `elevation-3` | modal, popover | ... | ... |
| `elevation-4` | toast, top-most | ... | ... |

[For glass strategy, additionally declare: blur levels (e.g., `blur-sm 12px`, `blur-md 20px`, `blur-lg 40px`), border alpha for edge definition, optional inner-shine convention.]

[For tonal-layers strategy, additionally declare: which surface token from the color system maps to each elevation level, so downstream agents don't reinvent the mapping.]

---

## Snippets

### Tailwind + shadcn CSS variables (primary)

```css
/* globals.css */
@layer base {
  :root {
    --bg: 0 0% 100%;
    --surface: 220 15% 98%;
    --text: 222 47% 11%;
    /* ... */
  }
  .dark {
    --bg: 222 15% 7%;
    --surface: 222 15% 11%;
    --text: 0 0% 96%;
    /* ... */
  }
}
```

```ts
// tailwind.config.ts (extend)
export default {
  theme: {
    extend: {
      colors: {
        bg: 'hsl(var(--bg))',
        surface: 'hsl(var(--surface))',
        text: 'hsl(var(--text))',
        // ...
      },
      borderRadius: { sm: '4px', md: '8px', lg: '12px', xl: '16px' },
      transitionDuration: { fast: '100ms', base: '150ms', slow: '250ms' },
      boxShadow: {
        'elevation-1': 'var(--elevation-1)',
        'elevation-2': 'var(--elevation-2)',
        'elevation-3': 'var(--elevation-3)',
        'elevation-4': 'var(--elevation-4)',
      },
    },
  },
}
```

### Alternative: theme.ts (CSS-in-JS)

```ts
// theme.ts
export const tokens = { /* same structure */ }
```

---

## Applied to codebase

**Kept as-is:** [list of tokens already present that we're not changing]
**Added:** [new tokens]
**Modified:** [tokens whose values changed — with rationale]

## Do's and Don'ts

Guardrails for downstream agents (`design-components`, `frontend-components`) and reviewers. At least 4 do's and 4 don'ts, tied to decisions actually encoded in the tokens above. Generic a11y advice doesn't count — each rule must reference a specific token, contrast pair, or elevation/motion choice from this file.

- **Do** use `accent` only for the single most important action per screen — every additional use dilutes its signal.
- **Do** pair every status color with its `-foreground` variant; never put raw `text` on `success`/`danger`/`warning`/`info` fills.
- **Do** cap elevation at `elevation-3` for persistent UI; reserve `elevation-4` for transient toasts only.
- **Do** use `motion.base` (150ms) for default transitions; `fast` for hover/press, `slow`/`slower` only for layout shifts.
- **Don't** introduce shadows, radii, or colors outside this file — if something feels missing, extend TOKENS.md rather than hardcoding.
- **Don't** mix elevation strategies (e.g., shadow cards next to glass cards) within the same view.
- **Don't** use `text-subtle` for body copy — it only clears AA for UI chrome / metadata.
- **Don't** animate background or color changes longer than `motion.slow` — it feels laggy.
```

### Step 9: Hand off

Tell the user: "Tokens specced. These are the only values downstream components will use. Next: `design-components` will define how each component consumes them."

## Rules

- Never write code into the user's source tree from this skill — only markdown + fenced snippets.
- Every color token must have a dark and light value declared, even if visually similar.
- Dark-first means dark palette is designed intentionally (not auto-inverted). Light palette is derived, not the other way around.
- If the codebase already has a token with the same semantic role but a different name, keep the existing name. Consistency beats purity.
- All color pairs must pass WCAG AA contrast for their intended use (4.5:1 body text, 3:1 large text / UI). Flag any that don't.
- Every TOKENS.md must include a Brand & Style preamble, an Elevation & depth section (with a declared strategy), and a Do's and Don'ts block — these drive downstream agents' judgment on edge cases and are non-negotiable.
- Elevation tokens must have dark AND light values — shadow opacity usually needs to increase in dark mode to remain visible.
- Read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1. If `designMode: shadcn-theme`, jump to the Shadcn-theme mode section at the end — skip Steps 5–7. The output is a slim TOKENS.md, not a full token system.

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

````markdown
# Design Tokens: [Feature Name] (shadcn-theme mode)

## Brand & Style

[2–3 sentences on aesthetic intent — tone, density, vibe. Even in shadcn mode the narrative helps downstream agents pick between shadcn variants (`default` vs `outline` vs `ghost`) based on feel.]

## Theme source
[tweakcn export paste / shadcn preset "<name>" / manual answers]

## CSS variables

```css
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
  --chart-1: ...; --chart-2: ...; --chart-3: ...; --chart-4: ...; --chart-5: ...;
  --sidebar: ...; --sidebar-foreground: ...;
  --sidebar-primary: ...; --sidebar-primary-foreground: ...;
  --sidebar-accent: ...; --sidebar-accent-foreground: ...;
  --sidebar-border: ...; --sidebar-ring: ...;
  --radius: 0.5rem;
}
.dark {
  --background: ...;
  /* ... dark overrides ... */
}
```

## Font stack
- Sans: Inter, system-ui, sans-serif
- Mono: JetBrains Mono, monospace

## Elevation & depth (minimal)

**Strategy:** shadow-based (shadcn default). Tailwind's `shadow-sm` / `shadow` / `shadow-md` / `shadow-lg` / `shadow-xl` map to elevations 1–4. If the brief calls for glass or tonal layers instead, switch to `designMode: custom-system` — shadcn-theme mode is not the right container for those strategies.

| Level | Tailwind class | Use |
|-------|----------------|-----|
| `elevation-0` | (none) | flat surfaces, inline chrome |
| `elevation-1` | `shadow-sm` | cards at rest, table rows |
| `elevation-2` | `shadow-md` | hover cards, dropdowns |
| `elevation-3` | `shadow-lg` | modals, popovers, sheets |
| `elevation-4` | `shadow-xl` | toasts |

## What this mode does NOT include
- Full spacing scale (shadcn uses Tailwind defaults)
- Typography scale (shadcn primitives set their own)
- Motion tokens (shadcn uses Tailwind transition defaults)
- Z-index scale (shadcn components manage internally)
- Custom shadow scale (uses Tailwind's defaults — if you need bespoke shadows, switch to `designMode: custom-system`)

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

## Do's and Don'ts

At least 3 do's and 3 don'ts tied to this theme's specific choices (radius, primary hue, dark-mode behavior). Generic shadcn advice doesn't count.

- **Do** use `--primary` for the single highest-priority action per screen; secondary actions go to `--secondary` or `ghost` variants.
- **Do** reach for `--muted-foreground` for metadata/timestamps; keep body copy on `--foreground`.
- **Do** use `shadow-md` or stronger only for overlays — resting UI sits at `shadow-sm` or flat.
- **Don't** introduce ad-hoc colors outside the CSS vars — extend `globals.css` or open a token question instead.
- **Don't** mix `--destructive` with neutral `--primary` in the same CTA group; destructive actions should be visually isolated.
- **Don't** change `--radius` per-component; shadcn primitives assume a single radius token.
````

### Hand off

"Tema shadcn specced. Próximo: `design-components` em modo shadcn-theme — ele vai mapear telas para componentes shadcn em vez de re-especificar primitivos."
