---
name: frontend-stack
description: Reads `.frontend/<feature>/FRONTEND_INTAKE.md` and (if present) `.design/<feature>/TOKENS.md`, and produces `.frontend/<feature>/FRONTEND_STACK.md` — the canonical decision document (framework, styling, state, forms, component strategy, auth client, testing, hosting), project structure, coding conventions, env vars, and decision log. Use when frontend-flow invokes it or when user says "define the frontend stack". Not for reopening design choices or for generic framework debates.
---

# Frontend Stack — Decision Document

Locks every frontend technology and pattern choice.

## How to run

### Step 1: Load intake

Read `.frontend/<feature>/FRONTEND_INTAKE.md`. If missing, direct user to run `frontend-brief-intake` first.

Read `.pm/<feature>/PROJECT_PROFILE.md` to get `designMode` and `uiFramework`. If missing, prompt the user once and create it (same migration prompt used by `pm-handoff` and `design-flow`).

Read `.qa/<feature>/QA_STRATEGY.md` if it exists. It declares testing tooling decisions that frontend-stack must be consistent with.

Also read `.design/<feature>/TOKENS.md` if present — it constrains styling choices.

### Step 2: Fill the 8-row decision table

| Concern | Typical options | Default |
|---------|-----------------|---------|
| Framework | Next.js App Router / Vite + React / Remix / SvelteKit | Next.js App Router |
| Styling | Tailwind + shadcn / vanilla-extract / stitches / CSS modules | Tailwind + shadcn |
| Server state | TanStack Query / SWR / native RSC / Apollo | TanStack Query |
| Client state | Zustand / Jotai / Redux Toolkit / Context | Zustand |
| Forms | react-hook-form + Zod / formik + yup / native | react-hook-form + Zod |
| Component library | shadcn/ui (customized) / Radix primitives / roll-own | shadcn/ui |
| Auth client | Clerk / next-auth / Supabase / custom | match backend |
| Testing | Vitest + Testing Library + Playwright / Jest + Cypress | Vitest + RTL + Playwright |
| Hosting | Vercel / Cloudflare Pages / Netlify / static + CDN | Vercel |

Every row needs rationale pointing to intake or design constraints.

### Step 2.5: Profile conflict check

Cross-check the Component library row against `PROJECT_PROFILE.md`:

- If `designMode: shadcn-theme` AND Component library ≠ `shadcn/ui` (customized) → **abort**:

  > "Conflito: PROJECT_PROFILE diz `shadcn-theme` mas você escolheu `[X]` como Component library. Reconcile antes de continuar: ou troque o Component library para `shadcn/ui`, ou edite `.pm/<feature>/PROJECT_PROFILE.md` para `designMode: custom-system`."

- If `designMode: custom-system` AND Component library = `shadcn/ui` → proceed, but note in the decision log that shadcn is being used as a primitive base under a custom token system (valid, just explicit).

### Step 2.6: QA tooling consistency check

If `.qa/<feature>/QA_STRATEGY.md` exists, cross-check the Testing row of the decision table against its tool choices:

- If frontend-stack picks a testing tool (e.g., Cypress) different from what QA_STRATEGY specified (e.g., Playwright) → **warn** in the decision log; do not abort:

  > "Nota: frontend-stack escolheu `[X]` para testes, mas QA_STRATEGY diz `[Y]`. Reconcile no decision log ou edite um dos dois. Não é bloqueante — ferramentas de teste são negociáveis."

Testing-tool conflicts are negotiable (dev and QA can reach alignment later); shadcn/ui is structural (Step 2.5 — aborts).

### Step 3: Coding conventions

**Component patterns:**
- Server Components by default (RSC), Client only when needed (`'use client'` with explicit justification)
- File: one component per file, named `PascalCase.tsx`, colocate styles if not Tailwind
- Props: explicit interface, no `any`, no prop-drilling beyond 2 levels (lift to context/store)

**Data fetching:**
- Server data: RSC + `fetch` (Next.js) or TanStack Query (Vite/SPA)
- Mutations: always via action / mutation hook, never inline `fetch` in handlers
- Optimistic updates: TanStack Query `onMutate` pattern where UX demands it

**State rules:**
- Zustand stores live under `src/stores/`, one per domain
- Never store server-derived data in client store — query it via TanStack Query
- Form state belongs to react-hook-form, never Zustand

**TypeScript:**
- `strict: true`, `noUncheckedIndexedAccess: true`
- No implicit `any`; prefer `unknown` and narrow
- Import paths via `@/*` alias

**Styling:**
- Tailwind utility classes grouped by concern (layout, typography, color, state) — use `cn()` util
- No inline styles except dynamic values (css vars)
- Design tokens referenced by name (`bg-accent`, `text-muted`) — never raw hex

**Accessibility baseline:**
- Every interactive element focusable, focus ring visible
- `<button>` for actions, `<a>` for navigation — never swap
- Labels for every form input (aria-label or `<label>`)
- Semantic HTML: heading levels sequential, no skipped levels

**Testing strategy:**
- Unit (Vitest + RTL): hooks, utilities, pure components
- Integration (Vitest + RTL): component + its state/server-state interactions
- E2E (Playwright): critical user journeys only (login, create-and-save, checkout)
- No visual regression in v1 (defer)

### Step 4: Canonical project structure

Adapt to chosen framework. Examples:

**Next.js App Router + Tailwind + shadcn:**

```
src/
├── app/
│   ├── (auth)/
│   │   └── login/page.tsx
│   ├── (app)/
│   │   ├── layout.tsx         # authenticated shell
│   │   ├── dashboard/page.tsx
│   │   └── projects/[id]/
│   │       ├── page.tsx
│   │       ├── loading.tsx
│   │       └── error.tsx
│   ├── api/
│   │   └── [...]/route.ts
│   └── layout.tsx             # root
├── components/
│   ├── ui/                    # shadcn primitives
│   └── feature/               # feature components
├── hooks/
├── stores/                    # zustand
├── lib/
│   ├── api/                   # generated from OpenAPI
│   └── utils.ts
├── schemas/                   # zod schemas
└── styles/globals.css
```

**Vite + React + Tailwind:**

```
src/
├── pages/
├── components/
│   ├── ui/
│   └── feature/
├── hooks/
├── stores/
├── lib/
├── router.tsx
├── main.tsx
└── App.tsx
```

### Step 5: Environment variables

| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| NEXT_PUBLIC_API_BASE_URL | yes | backend API URL | https://api.example.com |
| NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY | yes | Clerk public key | `pk_test_...` |
| CLERK_SECRET_KEY | yes | Clerk server key | `sk_test_...` |
| NEXT_PUBLIC_APP_URL | no | base URL for links | auto-detected in dev |

### Step 6: Write FRONTEND_STACK.md

Save to `.frontend/<feature>/FRONTEND_STACK.md`. Structure:

```markdown
# Frontend Stack: [Feature Name]

**Intake:** FRONTEND_INTAKE.md

## Stack decisions

| Concern | Choice | Version | Rationale |
|---------|--------|---------|-----------|
| Framework | Next.js App Router | 14.x | intake confirmed; RSC simplifies data fetching |
| Styling | Tailwind + shadcn | 3.x / 0.9 | matches design TOKENS.md format |
| Server state | TanStack Query | 5.x | cache invalidation + optimistic updates |
| Client state | Zustand | 4.x | tiny API, no boilerplate |
| Forms | react-hook-form + Zod | 7.x / 3.x | shared schemas with backend |
| Component lib | shadcn/ui | latest | matches COMPONENT_SPECS |
| Auth client | Clerk | latest | matches backend |
| Testing | Vitest + RTL + Playwright | latest | fast + standard |
| Hosting | Vercel | — | Next.js idiomatic |

## Project structure

[Tree]

## Coding conventions

### Component patterns
[Full detail]

### Data fetching
[...]

### State rules
[...]

### TypeScript
[...]

### Styling
[...]

### Accessibility baseline
[...]

### Testing strategy
[...]

## Environment variables

[Table]

## Local development setup

1. `git clone` + `npm ci`
2. `cp .env.example .env.local` and fill values
3. `npm run dev`
4. `npm test`, `npm run test:e2e` (requires backend running)

## Decision log

| # | Decision | Options | Chosen | Rationale |
|---|----------|---------|--------|-----------|
| 1 | Framework | Next / Remix / Vite | Next App Router | RSC + team XP |
| ... | | | | |
```

### Step 7: Hand off

Tell the user: "Stack locked. Next: `frontend-routing` will translate the design IA into concrete routes."

## Rules

- Every row in the stack table needs a rationale — not "it's popular" but why it fits THIS project.
- If the codebase already has a different choice, reconcile: either migrate (requires task in FRONTEND_TASKS) or keep what's there (update the stack doc to match reality).
- Component lib choice must be consistent with `TOKENS.md` format — picking Tailwind + shadcn aligns with TOKENS that are CSS variables.
- Coding conventions must be concrete enough for two devs to implement the same pattern independently.
- This skill does not write code — only decisions + conventions.
