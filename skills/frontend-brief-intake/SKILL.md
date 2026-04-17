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

Ask questions (one at a time, Critical → Important) **only for ⚠️ / ❌ items**. Offer defaults:

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
