---
name: frontend-routing
description: Reads `.design/<feature>/IA.md` and `.frontend/<feature>/FRONTEND_STACK.md` and produces `.frontend/<feature>/FRONTEND_ROUTES.md` — concrete routes with file paths, layouts, route groups, auth guards, middleware, and loading/error/not-found boundaries. Translates the design information architecture into the framework's routing model. Use when frontend-flow invokes it or when user says "plan the routing", "map IA to routes". Not for generic routing questions or routes unrelated to the feature at hand.
---

# Frontend Routing — IA-to-Routes Mapping

Translates the design `IA.md` into the framework-specific routing structure.

## How to run

### Step 1: Load inputs

Read:
- `.design/<feature>/IA.md` (sitemap, navigation model, boundaries — MUST exist)
- `.frontend/<feature>/FRONTEND_STACK.md` (for framework idiom)
- Existing codebase routes (from brief-intake audit)

If `IA.md` is missing, abort. Tell the user: "I need `.design/<feature>/IA.md` from design-flow. Run that first."

### Step 2: Map each IA route to a file

For Next.js App Router:
- `/` → `app/page.tsx`
- `/dashboard` (authenticated) → `app/(app)/dashboard/page.tsx`
- `/auth/login` (public) → `app/(auth)/login/page.tsx`
- `/projects/[id]` → `app/(app)/projects/[id]/page.tsx`
- `/projects/[id]/tasks/[taskId]` (modal intercept) → `app/(app)/projects/[id]/@modal/(.)tasks/[taskId]/page.tsx`

For Vite + React Router:
- Declarative `<Route>` tree or file-based via `react-router`

For SvelteKit:
- `src/routes/+page.svelte`, `src/routes/(app)/...`, etc.

### Step 3: Define layouts and route groups

For each layout from IA.md:
- **File path** in the framework idiom (e.g., Next `(app)/layout.tsx`)
- **Children** (which routes use this layout)
- **Shell contents** (nav, sidebar, footer, auth boundary)
- **Data fetching** done at layout level (e.g., current user)

### Step 4: Declare middleware and guards

- **Middleware** (Next: `middleware.ts`): redirect rules (unauthenticated → /auth/login), locale, canonical host
- **Auth guard** per route group: `(app)/layout.tsx` calls `requireAuth()` or uses Clerk's `<SignedIn>`
- **Role guards**: admin-only routes (e.g., `/admin/*`) require role check

### Step 5: Declare boundaries per route

For every route:
- `loading.tsx` / `<Suspense>` skeleton — what shows while data loads
- `error.tsx` — how errors render (inline vs full-page)
- `not-found.tsx` — 404 for invalid params (project not found)

### Step 6: Write FRONTEND_ROUTES.md

Save to `.frontend/<feature>/FRONTEND_ROUTES.md`:

````markdown
# Frontend Routes: [Feature Name]

**Stack:** FRONTEND_STACK.md ([Next.js App Router])
**IA:** .design/feature-x/IA.md

## Route table

| IA route | File | Layout | Auth | Notes |
|----------|------|--------|------|-------|
| / | app/page.tsx | marketing | public | landing; redirect to /dashboard if signed in |
| /dashboard | app/(app)/dashboard/page.tsx | app | user | server component; prefetch projects |
| /projects | app/(app)/projects/page.tsx | app | user | list |
| /projects/[id] | app/(app)/projects/[id]/page.tsx | app | user (member) | dynamic segment |
| /projects/[id]/tasks | app/(app)/projects/[id]/tasks/page.tsx | app | user (member) | nested |
| /auth/login | app/(auth)/login/page.tsx | auth | public | |
| /auth/signup | app/(auth)/signup/page.tsx | auth | public | |
| /settings | app/(app)/settings/page.tsx | app | user | |

## Layouts

### `app/layout.tsx` (root)
- Fonts, theme provider, toaster
- No auth boundary here

### `app/(marketing)/layout.tsx` (implicit via `/`)
- Header with login CTA
- Minimal — no app chrome

### `app/(app)/layout.tsx`
- `requireAuth()` — redirects to /auth/login if unauthed
- Renders: `<AppShell>` with sidebar (desktop) / bottom tab bar (mobile)
- Fetches: current user, notifications count

### `app/(auth)/layout.tsx`
- Centered card, no nav
- Redirects to /dashboard if already signed in

## Middleware

### `middleware.ts`

```ts
// Shape reference
import { clerkMiddleware } from '@clerk/nextjs/server';
export default clerkMiddleware((auth, req) => {
  // no-op: per-route guards happen in layouts
});
export const config = {
  matcher: ['/((?!_next|.*\\..*).*)'],
};
```

## Guards

| Route group | Guard | Redirects to |
|-------------|-------|--------------|
| `(app)/*` | `requireAuth()` in layout | /auth/login?next=... |
| `/admin/*` | `requireRole('admin')` | 403 page |

## Boundaries

| Route pattern | loading.tsx | error.tsx | not-found.tsx |
|---------------|-------------|-----------|---------------|
| `/projects/[id]` | ProjectSkeleton | inline ErrorBanner + retry | ProjectNotFound page |
| default | spinner | full-page ErrorFallback | global NotFoundPage |

## Navigation wiring

Primary nav (from IA):
- Desktop: `<AppSidebar>` in `(app)/layout.tsx` — items: Dashboard, Projects, Settings
- Mobile: `<BottomTabBar>` in `(app)/layout.tsx` — items: Home, Projects, Notifications, Me

## Deep-linking

- `/projects/[id]?tab=tasks` — tab state in query param, preserved on reload
- `/projects/[id]/tasks/[taskId]` — intercepted route; renders as modal over project detail on navigation, as dedicated page on hard reload

## File-tree diff vs current repo

**New:**
- `app/(app)/layout.tsx`
- `app/(app)/projects/page.tsx`
- `app/(app)/projects/[id]/page.tsx`
- `app/(app)/projects/[id]/loading.tsx`
- `app/(app)/projects/[id]/error.tsx`
- `app/(auth)/login/page.tsx`
- `middleware.ts`

**Modified:**
- `app/layout.tsx` — add ThemeProvider + Toaster

**Removed:**
- (none)
````

### Step 7: Hand off

Tell the user: "Routes mapped. [N] routes × [M] layouts. Next: `frontend-components` maps the design component specs to implementation plans."

## Rules

- Every IA route must map to a file in the chosen framework. No route "for later" — it either maps or is deferred explicitly.
- Auth guards live in layouts (Next / SvelteKit) or in route config (React Router). Do not repeat per page.
- Boundaries (loading / error / not-found) are mandatory for any route that fetches data. Flag routes without boundaries.
- Navigation is wired in one place (the layout), not repeated per page.
- Deep-linking is default — if a route is modal-only, justify in the notes column.
