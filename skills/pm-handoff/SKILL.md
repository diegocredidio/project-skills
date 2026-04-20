---
name: pm-handoff
description: Reads prior PM artifacts from `.pm/<feature>/` and produces four discipline-specific briefing files (`.design/<feature>/DESIGN_BRIEF.md`, `.backend/<feature>/BACKEND_BRIEF.md`, `.frontend/<feature>/FRONTEND_BRIEF.md`, `.qa/<feature>/QA_BRIEF.md`), then optionally invokes the in-package specialist flows (design-flow, backend-flow, frontend-flow, qa-flow). User picks which workstreams to start now vs. later. Use when user says "handoff", "start building", "generate briefings", "I want to start the backend/design/frontend/qa", "move to execution", or when pm-flow completes. Not for invoking external design-flow packages — this package ships its own design-flow.
---

# PM Handoff — Bridge to Specialist Skill-Sets

Take everything produced by the PM skills and translate it into targeted briefings for each discipline. Then invoke the right specialist skill-set for each workstream.

## How to run

### Step 1: Verify PM artifacts

Check that `.pm/<feature-name>/` contains at minimum:
- `PRD.md` — required
- `ARCHITECTURE.md` — required
- `WORKSTREAMS.md` — required
- `TASKS.md` — required

If any are missing, warn the user: "The [file] is missing. You can run `pm-[step]` to generate it, or continue with what's available."

Optional but useful: `GRILL_SUMMARY.md`, `REVIEW.md`, `INTAKE.md`

### Step 1.5: Read PROJECT_PROFILE.md

Read `.pm/<feature-name>/PROJECT_PROFILE.md` to get `designMode`, `uiFramework`, and `testingRigor`.

If the file is missing (legacy project pre-dating this convention), prompt once:

> "Este projeto não tem PROJECT_PROFILE.md completo.
> Modo de design: **shadcn-theme** ou **custom-system**?
> Rigor de teste: **mvp** ou **full**?
> (Respostas gravam em `.pm/<feature-name>/PROJECT_PROFILE.md` — afetam design-tokens, design-components, qa-strategy, qa-review.)"

If the file exists but is missing only one of the two fields (e.g., legacy project with `designMode` but no `testingRigor`), ask only for the missing field; preserve what's there.

Write the file with the user's answer and continue. Do NOT re-run pm-grill.

### Step 2: Ask which workstreams to hand off

> "Which workstreams do you want to start? You can pick one now and come back for the others later — or start all four."

Options:
- Design → gera `DESIGN_BRIEF.md` e invoca `design-flow`
- Backend → gera `BACKEND_BRIEF.md` e invoca `backend-flow`
- Frontend → gera `FRONTEND_BRIEF.md` e invoca `frontend-flow`
- QA → gera `QA_BRIEF.md` e invoca `qa-flow`
- CC Guardrails → bootstrap Claude Code agents via `cc-flow`
- All five
- "Just generate the briefings" → gera os quatro arquivos sem invocar nenhuma skill ainda

Se o usuário quiser apenas um, gera só o briefing daquele e para. Os outros ficam disponíveis para quando ele quiser rodar `backend-flow` ou `frontend-flow` diretamente — essas skills leem os arquivos do disco de forma independente.

### Step 3: Generate briefing documents

For each selected workstream, generate a focused briefing that extracts only what that discipline needs from the PM artifacts. Save to a discipline-specific folder.

---

#### DESIGN_BRIEF.md

Save to `.design/<feature-name>/DESIGN_BRIEF.md` — this is the canonical input for the in-package `design-flow` (and specifically the `design-brief-intake` skill that reads it first).

```markdown
# Design Brief: [Feature/Project Name]

**designMode:** [shadcn-theme | custom-system — from PROJECT_PROFILE.md]
**uiFramework:** [shadcn/ui | <other> | none — from PROJECT_PROFILE.md]

## Project context
[2-3 sentences from the PRD problem statement and solution]

## Target users
[From GRILL_SUMMARY — who they are, their technical level, their context]

## User journeys to design
[From PRD — each journey as a numbered list of steps.
Be specific: what the user sees, what they do, what happens next.]

### Journey 1: [Name]
1. [Step]
2. [Step]

### Journey 2: [Name]
...

## Screens / pages required
[From ARCHITECTURE.md frontend routes + WORKSTREAMS.md design deliverables]

| Screen | Route | Description | Priority |
|--------|-------|-------------|----------|
| [Name] | /path | [What it does] | Must / Should / Nice |

## States to design (per screen)
[For each screen: default, loading, empty, error, success]

## Functional requirements for design
[From PRD — only the FR-XXX items that affect visual design or interaction]

## Constraints
- **Existing design system:** [Yes/No — if yes, what components exist]
- **Responsive:** [Breakpoints required]
- **Accessibility:** [WCAG level, specific needs]
- **Brand/style:** [Any constraints from the grill or PRD]

## Integration points visible to users
[From ARCHITECTURE.md — external services that affect UX: payments, auth, maps, etc.]

## Tasks already defined (for reference)
[From TASKS.md — design tasks only, so design-flow knows what's already scoped]
```

---

#### BACKEND_BRIEF.md

Save to `.backend/<feature-name>/BACKEND_BRIEF.md`.

```markdown
# Backend Brief: [Feature/Project Name]

## What we're building
[1 paragraph from PRD — the backend perspective only]

## Stack confirmed
[From ARCHITECTURE.md]
- Language/runtime: [e.g., Node.js 20, Python 3.12, Go 1.22]
- Framework: [e.g., Fastify, FastAPI, Gin]
- Database: [e.g., PostgreSQL 16, MongoDB, SQLite]
- ORM/query layer: [e.g., Prisma, SQLAlchemy, GORM]
- Auth: [e.g., JWT + bcrypt, Auth0, Supabase Auth]
- Hosting target: [e.g., Railway, AWS Lambda, Fly.io]

## API surface to implement
[From ARCHITECTURE.md — full endpoint table]

| Method | Path | Description | Auth required | Maps to FR |
|--------|------|-------------|---------------|------------|
| POST | /api/v1/... | ... | Yes/No | FR-XXX |

## Data model
[From ARCHITECTURE.md — entities, fields, relationships]

## Business logic requirements
[From PRD functional requirements — only backend-relevant items]

## External integrations
[From ARCHITECTURE.md integration table — only backend-owned integrations]

## Performance and scale targets
[From PRD non-functional requirements]

## Security requirements
[Auth rules, input validation expectations, rate limiting, CORS policy]

## Tasks already defined (for reference)
[From TASKS.md — backend tasks only]

## Interface contracts with frontend
[From WORKSTREAMS.md API contracts table — request/response shapes]
```

---

#### FRONTEND_BRIEF.md

Save to `.frontend/<feature-name>/FRONTEND_BRIEF.md`.

```markdown
# Frontend Brief: [Feature/Project Name]

**designMode:** [shadcn-theme | custom-system — from PROJECT_PROFILE.md]
**uiFramework:** [shadcn/ui | <other> | none — from PROJECT_PROFILE.md]

## What we're building
[1 paragraph from PRD — the frontend perspective]

## Stack confirmed
[From ARCHITECTURE.md]
- Framework: [e.g., Next.js 14, React + Vite, SvelteKit]
- Styling: [e.g., Tailwind CSS, CSS Modules, styled-components]
- State management: [e.g., Zustand, TanStack Query, React Context]
- Component library: [e.g., shadcn/ui, Radix, custom]
- Auth client: [e.g., next-auth, Clerk, custom]
- Hosting: [e.g., Vercel, Cloudflare Pages, Netlify]

## Pages / routes to build
[From ARCHITECTURE.md frontend routes]

| Route | Component | Description | Auth required |
|-------|-----------|-------------|---------------|
| /path | PageName | [What it shows] | Yes/No |

## API endpoints to consume
[From WORKSTREAMS.md interface contracts + ARCHITECTURE.md — frontend perspective]

| Endpoint | Purpose | Trigger | Expected response |
|----------|---------|---------|------------------|
| POST /api/v1/... | [Why] | [User action] | [Shape] |

## State to manage
[From ARCHITECTURE.md — state management strategy, what lives where]

## Design handoff
[Will be available in `.design/<feature-name>/` — list expected design deliverables]

## Functional requirements for frontend
[From PRD — only FR-XXX items that affect the UI]

## Non-functional requirements
- Responsive breakpoints: [From PRD]
- Accessibility: [WCAG level, specifics]
- Performance targets: [e.g., LCP < 2.5s, INP < 200ms]

## Tasks already defined (for reference)
[From TASKS.md — frontend tasks only]
```

---

#### QA_BRIEF.md

Save to `.qa/<feature-name>/QA_BRIEF.md` — this is the canonical input for the in-package `qa-flow` (and specifically the `qa-brief-intake` skill that reads it first).

```markdown
# QA Brief: [Feature/Project Name]

**designMode:** [shadcn-theme | custom-system — from PROJECT_PROFILE.md]
**testingRigor:** [mvp | full — from PROJECT_PROFILE.md]
**uiFramework:** [shadcn/ui | <other> | none — from PROJECT_PROFILE.md]

## What we're testing
[1 paragraph from PRD — the QA perspective: what quality means for this feature, critical user outcomes that must not regress]

## Testing rigor selected
**testingRigor:** [mvp | full]

Implications:
- `mvp` → smoke pyramid, 1 E2E per critical journey, covers only FR-XXX marked "must", no formal a11y/perf/visual-regression matrix
- `full` → complete pyramid, explicit numeric coverage targets, every FR covered plus edge/negative cases, a11y matrix (WCAG AA per component), perf budgets, visual regression

## Functional requirements to test
[From PRD — list FR-XXX items with priority. In `mvp` mode the must-have set is the minimum scope; in `full` mode all FR-XXX must be covered.]

| FR ID | Priority | Description | Test approach |
|-------|----------|-------------|---------------|
| FR-XXX | Must / Should / Nice | [What it does] | Unit / Integration / E2E |

## User journeys to E2E
[From PRD user journeys — each journey becomes at least one E2E test case in `qa-cases`.]

### Journey 1: [Name]
1. [Step]
2. [Step]

### Journey 2: [Name]
...

## External integrations to test
[From ARCHITECTURE.md integration table — each external integration needs contract/mock strategy]

| Integration | Purpose | Test approach | Mock strategy |
|-------------|---------|---------------|---------------|
| [Service] | [Why] | Contract / Stubbed / Live | [How] |

## Non-functional testing targets
[Presence depends on testingRigor — in `mvp` only critical NFRs; in `full` the full matrix]

- **Performance:** [e.g., LCP < 2.5s, INP < 200ms, API p95 < 300ms — full mode specifies budgets per route/endpoint]
- **Accessibility:** [e.g., WCAG AA — full mode specifies matrix per component]
- **Security:** [Auth coverage, input validation tests, rate limiting tests]
- **Visual regression:** [full mode only — tool + baselines]

## Test environments and data strategy
- **Environments:** [local, CI, staging — which runs which tests]
- **Test data:** [fixtures, seeds, factories, production-like snapshots]
- **Data isolation:** [per-test teardown, transactional rollback, separate schema]
- **Secrets handling in tests:** [how auth/API keys are provided in CI]

## Tasks already defined (for reference)
[From TASKS.md — QA section, if present. If QA_TASKS.md does not yet exist, this will be TBD until `qa-flow` runs.]
```

---

### Step 4: Invoke specialist skill-sets

After generating each briefing, invoke the appropriate skill:

**Design:**
> "DESIGN_BRIEF.md is ready at `.design/<feature-name>/`. Now invoking `design-flow` — this runs the full design workflow (design-brief-intake → design-ia → design-tokens → design-components → design-tasks → design-review) using this brief as the starting point on Path A."
>
> Run: `design-flow`
> The `design-brief-intake` skill will read our DESIGN_BRIEF.md as its primary input and enrich it with a codebase audit.

**Backend:**
> "BACKEND_BRIEF.md is ready at `.backend/<feature-name>/`. Now invoking `backend-flow` — this runs the full backend workflow (backend-brief-intake → backend-stack → backend-data → backend-api → backend-tasks → backend-review)."
>
> Run: `backend-flow`

**Frontend:**
> "FRONTEND_BRIEF.md is ready at `.frontend/<feature-name>/`. Now invoking `frontend-flow` — this runs the full frontend workflow (frontend-brief-intake → frontend-stack → frontend-routing → frontend-components → frontend-tasks → frontend-review). It will also read any design artifacts available at `.design/<feature-name>/`."
>
> Run: `frontend-flow`

**QA:**
> "QA_BRIEF.md is ready at `.qa/<feature-name>/`. Now invoking `qa-flow` — this runs the full QA workflow (qa-brief-intake → qa-strategy → qa-cases → qa-tasks → qa-review)."
>
> Run: `qa-flow`
> `qa-brief-intake` reads QA_BRIEF.md as its primary input and audits existing test infrastructure.

**CC Guardrails:**
> "Planning artifacts available at `.pm/<feature-name>/` and the discipline folders. Now invoking `cc-flow` — on first run this copies starter templates into `.cc-templates/` and writes `.pm/cc-config.md`; then `cc-sync` renders per-feature agents into `.claude/agents/<feature-name>/` for the dev to invoke during implementation."
>
> Run: `cc-flow`

### Step 5: Coordination note

If all five are running in parallel, remind the user of the sequencing reality:

> "Design should start first — frontend needs the specs before building components. Backend can run in parallel with design. QA can start after PRD + architecture are done (in parallel with design/backend). Frontend waits on both design specs and backend contracts. Once everything is built, design-review, backend-review, frontend-review, and qa-review run in parallel. CC Guardrails (`cc-flow`) can run any time after the discipline artifacts exist — it doesn't block any other workstream and produces `.claude/agents/<feature>/` for developers to invoke during implementation."

## Rules

- Never generate the briefings from memory — always read the `.pm/` files fresh
- Each briefing must extract only what's relevant to that discipline — no noise from other workstreams
- The DESIGN_BRIEF.md format must be compatible with the in-package `design-brief-intake` skill input
- If a required piece of information is missing from PM artifacts, flag it in the briefing as `[TO BE DEFINED]` rather than inventing it
- The handoff is one-way — do not loop back into PM skills from here. If new requirements emerge during specialist work, use `pm-review` to reassess
- Read `.pm/<feature-name>/PROJECT_PROFILE.md` before generating briefs. If missing, prompt once and create it — do NOT re-run pm-grill.
- `cc-flow` is optional but closes the dev loop — invoke after all disciplines have produced artifacts and it writes guardrails into `.claude/agents/<feature>/` for developers to invoke.
