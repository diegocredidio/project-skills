---
name: frontend-brief-intake
description: Reads `.frontend/<feature>/FRONTEND_BRIEF.md` (from pm-handoff) and available design artifacts at `.design/<feature>/`, audits the frontend codebase, and runs an embedded gap-fill grill for anything ambiguous. Produces `.frontend/<feature>/FRONTEND_INTAKE.md`. Use when frontend-flow invokes it or when user says "intake the frontend brief", "audit the frontend". Not for generic "should I use React?" discussions or for replanning scope resolved by PM.
---

# Frontend Brief Intake — Brief + Design + Codebase + Gap-Fill Grill

Produces the intake document that `frontend-stack` and all downstream frontend skills read.

## How to run

### Step 1: Load inputs

Read:
- `.frontend/<feature>/FRONTEND_BRIEF.md` (from pm-handoff) OR fall back to `.pm/<feature>/ARCHITECTURE.md`
- `.design/<feature>/DESIGN_BRIEF.md` (if exists)
- `.design/<feature>/TOKENS.md` (if exists — influences styling-system choice)
- `.design/<feature>/IA.md` (if exists — influences routing strategy)
- `.design/<feature>/COMPONENT_SPECS.md` (if exists — influences component lib choice)
- `.design/<feature>/DESIGN_TASKS.md` (if exists — informs task dependencies)

Catalog which design artifacts are present / missing. The matrix goes into the output.

### Step 2: Audit the codebase

Read:
- `package.json` → framework (Next.js / Vite + React / Remix / SvelteKit / Solid / Vue), styling (Tailwind / CSS modules / styled-components / stitches / vanilla-extract), state (Zustand / Redux Toolkit / TanStack Query / none), forms (react-hook-form / zod-form / formik), testing (Vitest / Jest / Playwright / Cypress)
- Framework config: `next.config.*`, `vite.config.*`, `svelte.config.*`, `astro.config.*`
- `tailwind.config.*` and `globals.css` / `app.css`
- `tsconfig.json` for paths / strictness
- `app/` or `src/` tree for existing patterns (components/, hooks/, stores/, lib/)
- Existing routes: App Router `app/`, Pages Router `pages/`, Vite `src/pages/`, SvelteKit `src/routes/`

Record what's present with versions.

### Step 2.5: Parent artifact read (lineage-only)

If `.pm/<feature>/PARENT.md` exists, this feature is an evolution of a parent slug. Read the parent slug from `PARENT.md`'s H1 (`# Parent: <parent-slug>`), then also read:
- `.frontend/<parent-slug>/FRONTEND_STACK.md` — framework, styling, state, forms, auth client, testing, hosting
- `.frontend/<parent-slug>/FRONTEND_ROUTES.md` — route inventory, layout tree, auth gates
- `.frontend/<parent-slug>/COMPONENT_PLAN.md` — component inventory and per-component source (shadcn, custom, reuse)

Treat these as ✅ clear inheritance. In Step 3, classify the inherited concerns as ✅ clear — do NOT re-interrogate framework, styling system, state management, forms library, auth client, testing tools, or hosting when those were resolved in the parent. At emission time (the step that writes the intake document), annotate each inherited concern with "(inherited from parent)" in the Source column of the output's "What's clear" or equivalent table. The embedded gap-fill grill (Step 4) asks only about concerns the child INTRODUCES.

Record parent artifact paths in the output's "Parent baseline" section alongside the codebase-detected section.

If the parent folder is missing any of the three files, proceed with what's available and note the absence.

If `PARENT.md` does not exist, skip this step and continue to Step 3 in standalone mode.

### Step 2.6: BDD contract check

Check for `.bdd/<feature>/FRONTEND_FEATURES.md`.

If present:
- Read the file
- For each FR-ID in the file, note the behavioral contract (the scenarios that describe what the frontend must do from the user's perspective)
- These contracts are pre-agreed behavioral specifications — a frontend concern with a matching scenario in FRONTEND_FEATURES.md is behaviorally specified; classify it as ✅ clear in Step 3 (behavior is defined), not as a grill candidate
- Note the FR-IDs covered by BDD scenarios for use in the Step 4 grill gate

If absent: skip this step.

### Step 3: Classify every frontend concern

For each, mark ✅ clear / ⚠️ ambiguous / ❌ missing:

- Framework + version
- Styling system
- Component library strategy (shadcn / Radix primitives / headless UI / roll-own / build on top of existing)
- State management (server state: TanStack Query; client state: Zustand / Jotai / Context)
- Data-fetching (fetch / SWR / TanStack Query / RSC)
- Forms (library + validation)
- Auth client (Clerk SDK / next-auth / Supabase client / custom)
- Testing (unit: Vitest / Jest; e2e: Playwright / Cypress)
- i18n needs
- Analytics / telemetry
- Hosting / deploy target (Vercel / Cloudflare Pages / Netlify / static + CDN)

### Step 4: Embedded gap-fill grill

Ask questions (one at a time, Critical → Important) **only for ⚠️ / ❌ items**. Never re-ask ✅ items, never re-grill concerns that Step 2.5 classified as inherited from the parent, and never grill concerns for FRs that have matching scenarios in FRONTEND_FEATURES.md — the behavioral specification is already agreed. Offer defaults:

- "Framework — the codebase has Next.js 14 App Router. Confirm?"
- "Styling — Tailwind + shadcn (per design TOKENS.md)? Default yes if design used this."
- "State — TanStack Query for server state, Zustand for client state? Default yes."
- "Forms — react-hook-form + Zod? Default yes for most projects."
- "Auth client — matches backend choice (Clerk SDK if backend is Clerk)?"

Don't re-grill the design — if `TOKENS.md` exists and declares Tailwind, don't ask the user about styling system.

### Step 5: Reconcile conflicts

Surface conflicts between inputs:
- "Brief says Vite SPA but design TOKENS.md is authored in Tailwind CSS variables format that maps cleanly onto Tailwind config — both work. Tailwind runs on Vite fine."
- "Brief says 'SPA' but routing in IA.md has SSR-required meta tags. SPA won't work — choose: Next.js / Remix / keep SPA and drop meta requirement."

### Step 6: Write FRONTEND_INTAKE.md

Save to `.frontend/<feature>/FRONTEND_INTAKE.md`:

**Lineage-only sections:** sections marked with an HTML comment starting `<!-- lineage-only: ... -->` must be emitted ONLY when `.pm/<feature>/PARENT.md` exists. If lineage is absent, omit the entire section (comment and heading). When emitting, delete the HTML comment line — it's an authoring marker, not document content.

```markdown
# Frontend Intake: [Feature Name]

**Date:** [today]
**Brief:** FRONTEND_BRIEF.md
**Design artifacts available:** [✓ TOKENS, ✓ IA, ✓ COMPONENT_SPECS, ✗ DESIGN_TASKS]

## Decisions so far

| Concern | Value | Source |
|---------|-------|--------|
| Framework | Next.js 14 App Router | package.json + brief |
| Styling | Tailwind + shadcn | design TOKENS.md + codebase |
| ... | | |

## Ambiguities resolved in this intake

| Concern | Options | Chosen | Rationale |
|---------|---------|--------|-----------|
| Client state | Zustand / Jotai / Context | Zustand | smaller API, already in team familiarity |
| Forms | react-hook-form / formik | react-hook-form + Zod resolver | shared schemas w/ backend |

## Codebase reality

- **Framework detected:** Next.js 14.2 App Router
- **Styling:** Tailwind 3.4 + shadcn components in `components/ui/`
- **State:** none installed
- **Data fetching:** native `fetch` only — no TanStack Query yet
- **Testing:** none installed
- **Auth client:** none installed

<!-- lineage-only: emit this entire section ONLY when PARENT.md exists; delete this comment on emit -->
## Parent baseline

| Artifact | Path | Key inheritance |
|----------|------|-----------------|
| Frontend stack | .frontend/<parent-slug>/FRONTEND_STACK.md | [one-line summary of inherited framework/styling/state/forms/auth/testing] |
| Routes | .frontend/<parent-slug>/FRONTEND_ROUTES.md | [one-line summary of inherited routes and layout tree] |
| Components | .frontend/<parent-slug>/COMPONENT_PLAN.md | [one-line summary of inherited components and sourcing strategy] |

## Design artifact availability

| Artifact | Present | Path |
|----------|---------|------|
| TOKENS.md | yes | .design/feature-x/TOKENS.md |
| IA.md | yes | .design/feature-x/IA.md |
| COMPONENT_SPECS.md | yes | .design/feature-x/COMPONENT_SPECS.md |
| DESIGN_TASKS.md | missing | — |

## Conflicts

- [Brief said X, design said Y → resolution]

## Open questions for frontend-stack

- [Anything the grill couldn't resolve]
```

### Step 7: Hand off

Tell the user: "Intake done. [N] concerns classified, [M] ambiguities resolved. Design artifacts: [list]. Next: `frontend-stack` locks in framework + libraries + conventions."

## Rules

- Never re-ask decisions already resolved in the brief or design artifacts. If TOKENS.md says Tailwind, that's done.
- If a key design artifact is missing (TOKENS.md, COMPONENT_SPECS.md), proceed but flag it. Downstream skills handle the gap.
- Critical items (framework, styling, state) cannot be deferred — push to a decision or explicitly block.
- If brief + codebase conflict on framework, call it out immediately. Frontend stack choice cascades into everything.
- When `.pm/<feature>/PARENT.md` exists, inherit the parent's `FRONTEND_STACK.md`, `FRONTEND_ROUTES.md`, and `COMPONENT_PLAN.md` as ✅ clear. Never re-grill framework, styling system, state management, forms library, data-fetching approach, auth client, testing frameworks, or hosting target when the parent already resolved them. New UI concerns introduced by the evolution (new routes, new components) are still in scope for the grill.
- If `.bdd/<feature>/FRONTEND_FEATURES.md` exists, treat its scenarios as agreed behavioral contracts — do not re-derive or contradict them.
