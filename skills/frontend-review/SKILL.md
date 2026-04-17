---
name: frontend-review
description: Post-build review of the shipped frontend — a11y audit via axe-core + Playwright MCP, Lighthouse performance budget, responsive breakpoints, bundle size, hydration/runtime errors. Reads `.frontend/<feature>/FRONTEND_STACK.md`, `FRONTEND_ROUTES.md`, `COMPONENT_PLAN.md`, `FRONTEND_TASKS.md` and `.design/<feature>/TOKENS.md`. Produces `.frontend/<feature>/FRONTEND_REVIEW.md`. Defers pixel fidelity to `design-review` explicitly. Use when frontend-flow invokes it post-build or when user says "frontend review", "audit the frontend". Not for pre-build planning or for API correctness audit (that's backend-review).
---

# Frontend Review — Engineering-Quality Audit

Validates the implemented frontend on a11y, performance, responsive behavior, and bundle budget. Complements `design-review` (which validates visual fidelity).

## Pre-flight

Confirm:
1. A preview URL is running (local dev, staging, or prod).
2. Playwright MCP available for a11y / responsive capture. If not, abort.
3. The four frontend specs exist (STACK, ROUTES, COMPONENT_PLAN, TASKS).

## How to run

### Step 1: Load specs + routes

Read the four frontend specs. Extract route list from `FRONTEND_ROUTES.md`. Identify stack budgets from `FRONTEND_STACK.md` (if declared): e.g., "first-load JS < 200KB per route", "LCP < 2.5s".

Ask user for preview URL base (e.g., `http://localhost:3000`).

### Step 2: Smoke-test every route

For each route in `FRONTEND_ROUTES.md`:

- Navigate via Playwright MCP
- Capture console errors / warnings
- Capture network failures (4xx/5xx)
- Capture unhandled hydration mismatch warnings (Next/Remix apps)

### Step 3: A11y audit (axe-core)

For each route:
- Run `@axe-core/playwright` (or equivalent)
- Log every violation: rule, severity, target selector, route
- Classify severity: Critical (blocker) / Serious / Moderate / Minor

Common targets:
- Color contrast (uses TOKENS — if fails, may mean token drift between spec and implementation)
- Form labels
- Button discernible text
- Heading order
- Landmark regions
- Focus visible

### Step 4: Performance (Lighthouse or equivalent)

For each route (desktop + mobile profile):
- LCP, CLS, INP, TBT, FCP
- Compare to stack budgets (if declared); default budgets: LCP < 2.5s, CLS < 0.1, INP < 200ms

Flag anything worse than budget as Warning. >2× worse as Critical.

### Step 5: Responsive sweep

For each route, capture screenshots at 375 / 768 / 1280 widths.
Check:
- No horizontal scroll on 375
- Navigation collapses correctly (bottom nav on mobile per IA)
- Tap targets ≥ 44×44px on mobile
- Content reflows (no overlapping elements)

(Visual fidelity — colors, spacing, typography nuance — defers to `design-review`.)

### Step 6: Bundle size audit

- Run `next build` (or equivalent) and read route-level bundle report
- Report JS + CSS per route (first-load)
- Flag any route exceeding stack-declared budget
- Call out large unused deps (imports trees if tooling available)

### Step 7: Runtime errors

From smoke-test captures:
- Unhandled promise rejections
- React error boundaries activating
- Hydration mismatches (Next.js warnings)
- Memory leaks (console warnings about setState on unmounted components)

### Step 8: Write FRONTEND_REVIEW.md

Save to `.frontend/<feature>/FRONTEND_REVIEW.md`:

```markdown
# Frontend Review: [Feature Name]

**Date:** [today]
**Preview URL:** [url]
**Specs:** FRONTEND_STACK, FRONTEND_ROUTES, COMPONENT_PLAN, FRONTEND_TASKS
**Scope:** a11y + perf + responsive + bundle — does NOT cover pixel fidelity (see design-review)

## Executive summary

| Dimension | Score | Notes |
|-----------|-------|-------|
| A11y | FAIL (3 Critical) | contrast on focus ring, missing labels |
| Performance | CONCERN | LCP 3.1s on /projects/[id] (budget 2.5s) |
| Responsive | PASS | |
| Bundle | CONCERN | /dashboard first-load 287KB (budget 200KB) |
| Runtime stability | PASS | no unhandled errors |

## Findings

### Critical

**REVIEW-001 — Focus ring invisible in light mode on all routes**
- **Axis:** a11y / contrast
- **Route:** all
- **Evidence:** axe violation `color-contrast` on `:focus-visible` style; 2.7:1 contrast
- **Suggested fix:** bump `--focus-ring` in light mode or use `ring-offset-2` to contrast against background

**REVIEW-002 — Missing label on search input in header**
- **Axis:** a11y / forms
- **Route:** all (header)
- **Selector:** `input[type=search]` in `components/feature/header.tsx:42`
- **Suggested fix:** add `<label class="sr-only">` or `aria-label`

### Serious

**REVIEW-003 — /projects/[id] LCP 3.1s**
- **Axis:** performance
- **Cause:** hero image served unoptimized (3.2MB PNG)
- **Suggested fix:** use `next/image` with `priority` on hero; switch to WebP
- **Spec ref:** FRONTEND_STACK.md budget was 2.5s LCP

### Moderate

**REVIEW-004 — Bundle bloat: chart library unused**
- **Axis:** bundle
- **Route:** /dashboard (287KB first-load)
- **Cause:** `recharts` imported but the chart was removed in a previous iteration; tree-shaking didn't eliminate it due to side-effect in entry
- **Suggested fix:** remove `recharts` from deps

### Minor

...

## Coverage matrix

| Route | A11y violations | LCP | CLS | Bundle | Notes |
|-------|-----------------|-----|-----|--------|-------|
| / | 0 | 1.8s | 0.02 | 142KB | ✓ |
| /dashboard | 2 | 2.1s | 0.04 | 287KB (over budget) | ✗ bundle |
| /projects/[id] | 1 | 3.1s | 0.08 | 198KB | ✗ LCP |
| /auth/login | 0 | 1.5s | 0 | 112KB | ✓ |
| /settings | 3 | 1.9s | 0.03 | 145KB | ✗ a11y |

## Recommendations (ordered)

1. Fix REVIEW-001 (focus ring) — affects every focusable element app-wide.
2. Fix REVIEW-002 (label) — trivial fix, blocks WCAG AA.
3. Optimize hero on /projects/[id] — largest perf win.
4. Remove unused recharts dep — one-liner bundle save.

## Deferred to design-review

The following dimensions were NOT evaluated:
- Color/spacing/typography fidelity vs TOKENS.md
- Component variant correctness vs COMPONENT_SPECS.md
- Aesthetic judgment
- Motion/transition quality

Run `design-review` for those.
```

### Step 9: Summarize

Give a concise verbal summary:

> "Frontend review done. [N Critical / M Serious / K Moderate]. Top blocker: [REVIEW-001 focus ring]. Report saved. Design fidelity deferred to `design-review`."

## Rules

- Playwright MCP is mandatory. No fallback to "eyeball it".
- Every finding must cite a route, a selector / file, and a metric.
- Defer visual fidelity to `design-review` explicitly — do not critique colors or spacing as separate findings.
- Prioritization: Critical = blocks WCAG AA, breaks a route, or >2× over perf budget. Serious = degrades measurably. Moderate = opportunity to improve. Minor = style.
- Do not fix anything. Review only.
