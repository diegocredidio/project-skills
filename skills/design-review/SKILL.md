---
name: design-review
description: Post-build critique of an implemented design against `.design/<feature>/DESIGN_BRIEF.md`, `TOKENS.md`, and `COMPONENT_SPECS.md`. Captures screenshots via Playwright MCP across 3 breakpoints × 2 themes (dark/light) — mandatory, not optional. Produces `.design/<feature>/DESIGN_REVIEW.md` with prioritized Must / Should / Could fix list. Use when design-flow invokes it post-build or when user says "design review", "review the implementation", "critique the design". Not for pre-build critique (there's nothing to screenshot yet) or general code review (that's a different concern).
---

# Design Review — Post-Build Critique With Evidence

Evaluates the shipped UI against the design plan. Screenshots are non-negotiable — review without evidence is code review in disguise.

## Pre-flight

Before starting, confirm:
1. A preview URL is running (local dev server, staging, or production).
2. Playwright MCP is available. If not, try Cursor Browser MCP as fallback. If neither, **abort** — tell the user to install Playwright MCP and retry.
3. `.design/<feature>/DESIGN_BRIEF.md`, `TOKENS.md`, and `COMPONENT_SPECS.md` exist.

If any of the above fails, do not produce a review.

## How to run

### Step 1: Load inputs and route list

Read:
- `.design/<feature>/DESIGN_BRIEF.md` — intent
- `.design/<feature>/TOKENS.md` — what values should appear
- `.design/<feature>/COMPONENT_SPECS.md` — what variants/states should exist
- `.design/<feature>/IA.md` — routes to visit

Ask the user for the preview URL base (e.g., `http://localhost:3000`).

### Step 2: Capture screenshots

For each route in IA.md:

- **Breakpoints:** 375 (mobile), 768 (tablet), 1280 (desktop) — use Playwright `setViewportSize`
- **Themes:** dark and light — toggle between captures
- **States:** for each route, capture critical states listed in DESIGN_TASKS.md (e.g., loading, empty, error)

Save all screenshots to `.design/<feature>/screenshots/` with deterministic names:
`<route-slug>--<width>--<theme>--<state>.png`

Examples:
- `dashboard--375--dark--default.png`
- `dashboard--1280--light--empty.png`
- `projects-id--768--dark--error.png`

### Step 3: Apply the 10-axis rubric

Score each axis as PASS / CONCERN / FAIL with evidence:

1. **Visual hierarchy** — does the eye land on the primary action first?
2. **Token consistency** — all colors/spacing trace to TOKENS.md? no stray hex values?
3. **Aesthetic fidelity** — matches the tone/density declared in the brief?
4. **Component fidelity** — variants, states, and props match COMPONENT_SPECS.md?
5. **States coverage** — loading, empty, error, success present where specified?
6. **Responsive behavior** — 375 / 768 / 1280 all legible, no horizontal scroll, nav collapses correctly?
7. **Accessibility** — visible focus ring, contrast AA, tab order logical, ARIA where needed?
8. **Typography** — scale consistent, line-heights readable, no orphans/widows in critical copy?
9. **Dark mode quality** — not a simple invert; surfaces feel intentional?
10. **Mobile-first feel** — thumb zones respected, tap targets ≥ 44px, no hover-only interactions?

### Step 4: Prioritize findings

- **Must fix** — blocks ship. Breaks a declared spec, fails a11y AA, breaks at a target breakpoint.
- **Should fix** — before next iteration. Polish misses, minor inconsistency.
- **Could improve** — nice-to-have, defer.

Each finding needs:
- Axis (from the 10 above)
- Route + breakpoint + theme + state
- Screenshot filename
- File path + line number (if the code is at hand) — or "unknown" if not
- Suggested patch (short code snippet or token swap)

### Step 5: Write DESIGN_REVIEW.md

Save to `.design/<feature>/DESIGN_REVIEW.md`:

```markdown
# Design Review: [Feature Name]

**Date:** [today]
**Reviewer:** design-review skill
**Preview URL:** [url]
**Playwright MCP:** [yes / fallback to cursor-browser]

## Summary

| Rubric axis | Score |
|-------------|-------|
| Visual hierarchy | PASS |
| Token consistency | CONCERN (3 stray values) |
| Aesthetic fidelity | PASS |
| Component fidelity | FAIL (Modal missing destructive variant) |
| States coverage | CONCERN (empty state missing on /projects) |
| Responsive | PASS |
| Accessibility | CONCERN (focus ring invisible in light mode) |
| Typography | PASS |
| Dark mode | PASS |
| Mobile-first | PASS |

## Screenshots captured

| Route | Breakpoints | Themes | States | Files |
|-------|-------------|--------|--------|-------|
| /dashboard | 375, 768, 1280 | dark, light | default, empty | screenshots/dashboard--*.png |
| /projects/[id] | 375, 768, 1280 | dark, light | default, loading, error | screenshots/projects-id--*.png |

Total: [N] screenshots.

## Findings

### Must fix

**FIX-001 — Modal missing `destructive` variant**
- Axis: Component fidelity
- Route / breakpoint / theme / state: /projects/[id] / 1280 / dark / delete-confirm
- Screenshot: `screenshots/projects-id--1280--dark--delete-confirm.png`
- File: `components/ui/modal.tsx:42`
- Spec: COMPONENT_SPECS.md Modal variants should include `destructive`
- Suggested patch: add `variant?: 'default' | 'destructive'` prop and conditional header/button styling

**FIX-002 — Focus ring invisible in light mode**
- Axis: Accessibility
- Route: all
- Screenshot: `screenshots/dashboard--1280--light--default.png` (Tab key pressed)
- Suggested patch: `--focus-ring` variable in light mode is too close to `--bg`; bump contrast in `globals.css :root`

### Should fix

...

### Could improve

...

## What works

- [Positive observation 1]
- [Positive observation 2]
```

### Step 6: Summary to user

Give a concise verbal summary:

> "Review complete. [N Must] / [M Should] / [K Could]. Biggest blocker: [top Must]. Report saved to DESIGN_REVIEW.md with [T] screenshots."

## Rules

- Screenshots are non-optional. If capture fails for any reason, abort and tell the user why.
- Mandatory captures: 3 breakpoints × 2 themes × N routes — skip only if a route genuinely does not exist on a breakpoint (uncommon).
- Every finding must cite a screenshot. No "looks off" without evidence.
- Match findings to declared specs — "I think the padding is wrong" is not a finding. "TOKENS.md spacing-4 is 16px but Dashboard uses 12px for section padding" is.
- Prioritization is conservative: default to Should when unsure, not Must. Over-flagging trains people to ignore the report.
- Do not fix anything. Review only.
