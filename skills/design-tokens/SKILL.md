---
name: design-tokens
description: Reads `.design/<feature>/DESIGN_BRIEF.md` and `CODEBASE_AUDIT.md` and produces `.design/<feature>/TOKENS.md` — the canonical visual token system (color scales, spacing, typography, radius, motion, breakpoints). Dark-first by default, light as variation. Emits spec in markdown plus fenced code snippets in the project's format (CSS vars / Tailwind config / theme.ts) — does not write code into the user's source tree. Use when design-flow invokes it or when user says "design tokens", "generate the token system", "build the color palette". Not for component-level styling decisions (that's design-components) or general color-theory consulting.
---

# Design Tokens — Visual System Specification

Generates the semantic token layer that every downstream UI decision will reference.

## How to run

### Step 1: Load inputs

Read:
- `.design/<feature>/DESIGN_BRIEF.md` (aesthetic direction, tone, density)
- `.design/<feature>/CODEBASE_AUDIT.md` (existing tokens to respect)
- Optionally `.design/<feature>/IA.md` (to anticipate screens that need tokens)

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

### Step 7: Build radius, motion, breakpoints

- **Radius:** `none (0), sm (4), md (8), lg (12), xl (16), 2xl (24), full (9999)`
- **Motion durations:** `fast (100ms), base (150ms), slow (250ms), slower (400ms)`
- **Motion easing:** `ease-out (default), ease-in-out (layout), spring` — define `cubic-bezier` values
- **Breakpoints:** `sm 375, md 768, lg 1024, xl 1280, 2xl 1440` (mobile-first min-width queries)
- **Z-index:** `base (0), sticky (10), dropdown (20), overlay (30), modal (40), popover (50), toast (60)`

### Step 8: Write TOKENS.md

Save to `.design/<feature>/TOKENS.md`:

```markdown
# Design Tokens: [Feature Name]

**Target format:** [Tailwind + shadcn CSS vars / theme.ts / plain CSS]
**Dark mode:** dark-first, light as variation

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
```

### Step 9: Hand off

Tell the user: "Tokens specced. These are the only values downstream components will use. Next: `design-components` will define how each component consumes them."

## Rules

- Never write code into the user's source tree from this skill — only markdown + fenced snippets.
- Every color token must have a dark and light value declared, even if visually similar.
- Dark-first means dark palette is designed intentionally (not auto-inverted). Light palette is derived, not the other way around.
- If the codebase already has a token with the same semantic role but a different name, keep the existing name. Consistency beats purity.
- All color pairs must pass WCAG AA contrast for their intended use (4.5:1 body text, 3:1 large text / UI). Flag any that don't.
