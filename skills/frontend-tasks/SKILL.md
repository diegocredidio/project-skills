---
name: frontend-tasks
description: Reads `.frontend/<feature>/FRONTEND_STACK.md`, `FRONTEND_ROUTES.md`, and `COMPONENT_PLAN.md` plus `.design/<feature>/DESIGN_TASKS.md` and produces `.frontend/<feature>/FRONTEND_TASKS.md` — dependency-ordered vertical slices that consume backend API contracts and design artifacts. Each task references required DESIGN_TASK_XXX IDs as dependencies. Use when frontend-flow invokes it or when user says "frontend tasks", "break down the frontend work". Not for writing code, not for generic task management.
---

# Frontend Tasks — Vertical-Sliced Implementation Plan

Turns the frontend plan into a sequenced work list. Cross-references design task dependencies.

## How to run

### Step 1: Load inputs

Read:
- `.frontend/<feature>/FRONTEND_STACK.md`
- `.frontend/<feature>/FRONTEND_ROUTES.md`
- `.frontend/<feature>/COMPONENT_PLAN.md`
- `.design/<feature>/DESIGN_TASKS.md` (for dependencies)
- `.backend/<feature>/BACKEND_API.md` (endpoint contracts to consume)

Abort if STACK / ROUTES / COMPONENT_PLAN are missing. Warn but proceed if DESIGN_TASKS or BACKEND_API missing (flag dependencies as "pending").

### Step 2: Group into phases

1. **Foundation** — install deps, apply tokens (or confirm already applied via design-tasks), base layouts, theme provider
2. **Core routes** — one task per primary route from FRONTEND_ROUTES.md
3. **Data wiring** — TanStack Query setup, API client from BACKEND_API.md, auth provider
4. **States + interactions** — loading/error/empty per route
5. **Responsive + a11y polish**
6. **Review** — hand off to `frontend-review`

### Step 3: Write each slice as a vertical task

A slice = one user-visible outcome (a route reachable, a form submittable, a list loading). Not "build all components then wire data" — one task does structure + wiring + state.

Per task:
- **Title** (imperative)
- **Description**
- **Done when** (observable acceptance)
- **Routes touched** (from FRONTEND_ROUTES.md)
- **Components implemented/consumed** (from COMPONENT_PLAN.md)
- **API endpoints wired** (from BACKEND_API.md — specific endpoint keys)
- **States handled** (loading / error / empty / success)
- **Tests written** (unit / behavior / e2e)
- **Depends on** prior FRONTEND_TASK IDs + `DESIGN_TASK_XXX` IDs + `BACKEND_TASK_XXX` IDs
- **Size** (S / M / L)

### Step 4: Explicit design dependencies

Never re-order design work. If a frontend slice needs `Button` and `ProjectCard` components, it declares:
`requires: DESIGN_TASK_004 (Button specced), DESIGN_TASK_007 (ProjectCard specced)`

If a design task is missing, the frontend slice is **blocked** — flag it, don't work around.

### Step 5: Write FRONTEND_TASKS.md

Save to `.frontend/<feature>/FRONTEND_TASKS.md`:

```markdown
# Frontend Tasks: [Feature Name]

**Stack:** FRONTEND_STACK.md
**Routes:** FRONTEND_ROUTES.md
**Components:** COMPONENT_PLAN.md
**Design tasks:** .design/feature-x/DESIGN_TASKS.md
**Backend API:** .backend/feature-x/BACKEND_API.md

## Summary

| Phase | Count | Effort |
|-------|-------|--------|
| Foundation | 4 | ~4 days |
| Core routes | 5 | ~10 days |
| Data wiring | 2 | ~3 days |
| States + interactions | 4 | ~5 days |
| Responsive + a11y | 3 | ~4 days |
| Review | 1 | 1 day |

## Phase 1 — Foundation

### FRONTEND_TASK_001 — Scaffold project

**Description:** Confirm Next.js 14 App Router is set up per FRONTEND_STACK.md §3. Install deps declared in STACK (shadcn CLI init, TanStack Query, Zustand, react-hook-form, Zod, Clerk, Vitest, Playwright).
**Done when:**
- `npm run dev` serves root page without errors
- `npm test` passes with zero tests (config valid)
- `npm run test:e2e` runs Playwright smoke test for `/`
**Depends on:** —
**Size:** M

### FRONTEND_TASK_002 — Apply design tokens

**Description:** Install TOKENS.md values into `globals.css` and `tailwind.config.ts` per STACK.
**Done when:**
- `dark` and `:root` CSS variables set
- Tailwind utilities resolve to tokens (`bg-accent` → accent variable)
- Manual check: Button with `bg-accent` renders correctly in both themes
**Depends on:** 001, DESIGN_TASK_001 (tokens applied to codebase — can coordinate)
**Size:** S

### FRONTEND_TASK_003 — Build root + (app) layouts

**Description:** Implement `app/layout.tsx` with ThemeProvider + Toaster + fonts. Implement `app/(app)/layout.tsx` with auth guard + AppShell.
**Done when:**
- Root layout renders children with theme working
- `/dashboard` redirects to `/auth/login` when unauthenticated
- After login, `/dashboard` renders inside AppShell with sidebar (desktop) / bottom tab bar (mobile)
**Routes touched:** all
**Components consumed:** AppShell, BottomTabBar, Sidebar (from COMPONENT_PLAN)
**Depends on:** 001, 002, DESIGN_TASK_003 (PageShell specced)
**Size:** L

### FRONTEND_TASK_004 — Auth client setup (Clerk)

**Description:** Install Clerk SDK, wire middleware, login/signup pages.
**Done when:**
- `/auth/login` renders Clerk's `<SignIn>` component in our auth layout
- After sign-in, user redirects to `/dashboard` (or `?next=` param)
- `useUser()` returns the current user inside (app) routes
**Depends on:** 001, 003
**Size:** M

## Phase 2 — Core routes

### FRONTEND_TASK_005 — Dashboard route

**Description:** Implement `/dashboard` server component. Fetch current user projects via server-side fetch to `/v1/projects?limit=10`. Render list of ProjectCard + EmptyState when none.
**Done when:**
- Authenticated user sees their projects sorted by most recent
- Empty user sees EmptyState with CTA to create project
- Clicking ProjectCard navigates to `/projects/[id]`
- Loading state shows ProjectSkeleton
**Routes touched:** `/dashboard`
**Components consumed:** ProjectCard, EmptyState, ProjectSkeleton
**API wired:** GET /v1/projects
**States:** loading, empty, success
**Depends on:** 003, 004, DESIGN_TASK_004 (Dashboard mocked), DESIGN_TASK_005 (ProjectCard), BACKEND_TASK_006 (endpoint live)
**Size:** M

### FRONTEND_TASK_006 — Create project flow

**Description:** Implement `/projects/new` route with react-hook-form + Zod. POST to `/v1/projects`. Redirect to `/projects/[id]` on success.
**Done when:**
- Form validates name + slug (Zod schema shared with backend)
- Submit creates project and redirects
- Server error (409 slug conflict) shows inline field error
**Routes touched:** `/projects/new`
**API wired:** POST /v1/projects
**States:** idle, submitting, error
**Depends on:** 005, DESIGN_TASK_006, BACKEND_TASK_006
**Size:** M

### FRONTEND_TASK_007 — Project detail + task list

...

## Phase 3 — Data wiring

### FRONTEND_TASK_010 — TanStack Query setup + API client from OpenAPI

**Description:** Generate typed API client from `.backend/<feature>/BACKEND_API.md` OpenAPI spec (via `openapi-typescript`). Wire TanStack Query provider.
**Done when:** typed client exists under `src/lib/api/`; example hook `useProjects()` returns typed data.
**Depends on:** 001, BACKEND_TASK_006 (API available)
**Size:** M

## Phase 4 — States + interactions

### FRONTEND_TASK_012 — Loading and error boundaries

**Description:** Per FRONTEND_ROUTES.md boundaries table: `loading.tsx`, `error.tsx`, `not-found.tsx` for every route with data fetching.
**Done when:** simulated API delay shows loading; simulated 500 shows error; invalid `projectId` shows not-found.
**Depends on:** 005, 006, 007
**Size:** M

## Phase 5 — Responsive + a11y polish

### FRONTEND_TASK_015 — Breakpoint sweep

**Description:** Walk every route at 375/768/1280; fix layout breaks.
**Done when:** all routes legible at 3 breakpoints × 2 themes with no horizontal scroll.
**Depends on:** all core routes
**Size:** M

### FRONTEND_TASK_016 — Keyboard + focus management

**Description:** Tab through every route, confirm focus ring visible, Modal focus trap works, Escape closes overlays.
**Done when:** axe-core finds zero focus-related violations.
**Depends on:** all core routes
**Size:** M

## Phase 6 — Review

### FRONTEND_TASK_018 — Frontend review

**Description:** Invoke `frontend-review` with preview URL. Produces FRONTEND_REVIEW.md with a11y + perf + responsive + bundle findings.
**Depends on:** all prior deployed to preview
**Size:** S

## Dependency graph

```
001 ─┬─ 002 ─ 003 ─┬─ 004 ─ 005 ─ 006 ─ 007 ─ 010 ─ 012 ─ 015 ─ 018
     │            └──────────────────────┘                  ├─ 016
     └──(parallel)─ 010 ─────────────────┘
```

Design dependencies (cross-skill):
- 002 requires DESIGN_TASK_001 (tokens applied)
- 003 requires DESIGN_TASK_003 (PageShell)
- 005 requires DESIGN_TASK_004, DESIGN_TASK_005
- 006 requires DESIGN_TASK_006
- ...

Backend dependencies:
- 005 requires BACKEND_TASK_006 (GET /v1/projects live)
- 006 requires BACKEND_TASK_006 (POST /v1/projects live)
- 007 requires BACKEND_TASK_008 (Tasks CRUD live)

## Parallelizable

- 005 ∥ 006 ∥ 007 (different routes, different state)
- 015 ∥ 016 (different audit dimensions)
```

### Step 6: Hand off

Tell the user: "Frontend tasks ordered ([N] total, [P] parallelizable). Cross-discipline dependencies flagged. Hand to dev team. Return for `frontend-review` once preview is running."

## Rules

- Every task is a vertical slice — route + component + data wire + states + tests.
- Never re-order or restate design work. Declare `requires: DESIGN_TASK_XXX` — that's the only coupling.
- Tasks with unmet design or backend dependencies are **blocked**, not worked-around. Flag them explicitly.
- "Done when" criteria must be observable without reading source code — a user action, a visible result, a test passing.
- This skill does not write code. Implementation happens outside.
