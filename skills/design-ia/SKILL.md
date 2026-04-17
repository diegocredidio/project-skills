---
name: design-ia
description: Reads the enriched `.design/<feature>/DESIGN_BRIEF.md` and produces `.design/<feature>/IA.md` — sitemap, navigation model, URL strategy, content hierarchy, and component reuse map. Audits existing routing before proposing structure. Use when design-flow invokes it or when user says "design the IA", "map the navigation", "sitemap this feature". Not for visual design decisions (those belong in design-tokens / design-components) or generic navigation consulting.
---

# Design IA — Information Architecture

Translates user journeys from the brief into a concrete navigation and routing structure.

## How to run

### Step 1: Load inputs

Read `.design/<feature>/DESIGN_BRIEF.md`. Extract:
- User journeys (step-by-step flows)
- Screens / pages required
- Navigation expectations (if any)
- Auth-gated vs public content

Optionally read `.design/<feature>/CODEBASE_AUDIT.md` for existing routing patterns.

### Step 2: Audit existing routing

Scan the target codebase:
- **Next.js App Router:** `app/` tree, `layout.tsx`, route groups `(...)`
- **Next.js Pages Router:** `pages/` tree
- **Vite + React Router:** route configuration
- **SvelteKit:** `src/routes/` tree
- **Remix:** `app/routes/`
- **Plain React:** any `<Routes>` declarations

Record: existing top-level routes, existing layouts, auth guards already in place.

### Step 3: Map journeys to routes

For each user journey in the brief, translate steps into URL patterns. Be explicit about:
- Path pattern (`/projects/[id]/tasks`)
- Dynamic segments
- Query params vs path params
- Auth state required (public / authenticated / role-gated)
- Deep-linkable (can be bookmarked / shared)

### Step 4: Define the navigation model

Decide how routes surface in the UI:
- **Primary navigation** — top-level, always visible (home, search, primary feature)
- **Secondary navigation** — contextual within a section (tabs, side rail)
- **Utility navigation** — account, settings, notifications, search (often top-right)
- **Mobile navigation model** — bottom tab bar / drawer / hamburger

Per mobile breakpoint, what collapses and how.

### Step 5: Define layouts and route groups

Identify reusable layout shells:
- Shared outer layout (nav + footer)
- Authenticated vs public layouts
- Full-bleed vs contained layouts (onboarding, marketing, auth)

Map each route to a layout.

### Step 6: Define boundaries

Per route, declare:
- **Loading state** — skeleton / spinner / progress
- **Error boundary** — full-page error vs inline
- **Not-found** — global 404 or contextual

### Step 7: Component reuse map

Cross-reference the component inventory in DESIGN_BRIEF.md with the routes. Per component, list which routes consume it (helps `design-components` order the spec work).

### Step 8: Write IA.md

Save to `.design/<feature>/IA.md`:

```markdown
# Information Architecture: [Feature Name]

## Sitemap

```
/
├── /dashboard                [authenticated]
├── /projects
│   ├── /projects/[id]        [authenticated]
│   └── /projects/[id]/tasks  [authenticated]
├── /settings                 [authenticated]
└── /auth
    ├── /auth/login           [public]
    └── /auth/signup          [public]
```

## Route table

| Path | Auth | Layout | Notes |
|------|------|--------|-------|
| / | public | marketing | Landing |
| /dashboard | authenticated | app | Redirects from / for logged-in |
| /projects/[id] | authenticated | app | Dynamic segment = project id |
| ... | | | |

## Navigation model

- Primary (desktop): `[ Dashboard | Projects | Settings ]` — top
- Primary (mobile, < 768): bottom tab bar `[ Home | Projects | Notifications | Me ]`
- Secondary: tabs within `/projects/[id]` — `[ Overview | Tasks | Members | Settings ]`
- Utility: `[ Search | Notifications | Account ]` — top-right, desktop only

## Layouts

| Layout | Used by | Contents |
|--------|---------|----------|
| `marketing` | `/` | minimal header, wide hero container, footer |
| `app` | `/dashboard`, `/projects/*`, `/settings` | left rail (desktop) / bottom nav (mobile) + content |
| `auth` | `/auth/*` | centered card, no nav |

## Boundaries

| Route pattern | Loading | Error | Not-found |
|---------------|---------|-------|-----------|
| `/projects/[id]` | project skeleton | inline error banner | project-not-found page |
| default | spinner | full-page error | global 404 |

## Component reuse map

| Component | Consumed by |
|-----------|-------------|
| `ProjectCard` | `/dashboard`, `/projects` |
| `TaskList` | `/projects/[id]/tasks`, `/dashboard` |
| `EmptyState` | everywhere |
| `AuthForm` | `/auth/login`, `/auth/signup` |

## Deep-linking

- Share link for project view: `/projects/[id]`
- Filter state in query params (e.g., `?status=open`) — survives reload and is shareable
- Modal routes: `/projects/[id]/tasks/[taskId]` (use route groups to keep parent visible)
```

### Step 9: Hand off

Tell the user: "IA recorded with [N] routes and [M] layouts. Next: `design-tokens` will produce the visual system these routes will use."

## Rules

- Always audit existing routing before proposing — don't propose a `/projects` route if one already exists with a different shape; reconcile instead.
- Every route needs an auth declaration — "public" is a valid answer but never leave it unspecified.
- Navigation model must have a mobile answer. Default to bottom tab bar for 3–5 primary items, hamburger for 6+.
- Deep-linking is the default — any route a user might share should survive a reload. Only make something modal-only when there's a good reason.
