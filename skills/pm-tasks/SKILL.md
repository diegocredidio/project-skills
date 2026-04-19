---
name: pm-tasks
description: Reads prior PM artifacts from `.pm/<feature>/WORKSTREAMS.md` and produces `.pm/<feature>/TASKS.md` — dependency-ordered, vertical-slice tasks per workstream. Use when user says "create tasks", "break into tasks", "generate the task list", "what should we build first", or when the pm-flow orchestrator triggers this step. Not for generic task-management discussion, personal todos, or non-project work.
---

# PM Tasks — Task Generation

Break workstreams into ordered, dependency-aware, independently deliverable tasks. Each task should be small enough for one person to complete in 1-3 days.

## How to run

### Step 1: Gather inputs

Read ALL previous artifacts:
1. `.pm/<feature-name>/PRD.md` — user stories and requirements
2. `.pm/<feature-name>/ARCHITECTURE.md` — technical decisions
3. `.pm/<feature-name>/WORKSTREAMS.md` — scope per discipline
4. Explore the codebase — understand what already exists

### Step 2: Identify vertical slices

Don't create horizontal tasks ("build all models", "build all APIs", "build all UI"). Instead, identify vertical slices — thin, end-to-end features that touch multiple layers:

Example of BAD horizontal tasks:
- ❌ Create all database tables
- ❌ Build all API endpoints
- ❌ Build all frontend pages

Example of GOOD vertical slices:
- ✅ User can create an account (schema + API + UI + tests)
- ✅ User can view their dashboard (query + API + page + loading state)
- ✅ Admin can export data as CSV (query + endpoint + download button)

### Step 3: Order by dependency

Build the dependency graph:
1. **Foundation tasks first** — database schema, auth setup, project scaffold, design tokens
2. **Core user journeys** — the primary happy paths
3. **Secondary features** — supporting features that depend on the core
4. **Polish and edge cases** — error handling, empty states, responsive, a11y

### Step 4: Write the tasks document

```markdown
# Tasks: [Feature/Project Name]

**Based on:** PRD v[X], Architecture v[X], Workstreams v[X]
**Date:** [Today]
**Total tasks:** [N]

---

## Task Legend

- 🔴 Blocked — cannot start until dependency completes
- 🟡 Ready — dependencies met, can be picked up
- 🟢 In progress
- ✅ Done

---

## Phase 1: Foundation

### TASK-001: [Task title]
- **Workstream:** [Backend / Frontend / Design]
- **Description:** [What to build, specifically]
- **Acceptance criteria:**
  - [ ] [Specific, testable criterion]
  - [ ] [Specific, testable criterion]
- **Maps to:** FR-001, FR-002
- **Depends on:** None
- **Estimated effort:** [S / M / L]
- **Status:** 🟡 Ready

### TASK-002: [Task title]
- **Workstream:** [Backend / Frontend / Design]
- **Description:** [What to build]
- **Acceptance criteria:**
  - [ ] [Criterion]
- **Maps to:** FR-003
- **Depends on:** TASK-001
- **Estimated effort:** [S / M / L]
- **Status:** 🔴 Blocked

---

## Phase 2: Core User Journeys

### TASK-010: [Task title — vertical slice name]
- **Workstream:** [Multiple — note which parts]
- **Description:** [What to build end-to-end]
- **Sub-tasks:**
  - [ ] [Backend] Create [model/endpoint]
  - [ ] [Frontend] Build [component/page]
  - [ ] [Design] Define [states/interactions]
- **Acceptance criteria:**
  - [ ] User can [action] and sees [result]
  - [ ] Error state: when [condition], user sees [message]
  - [ ] Loading state displays while [operation] is in progress
- **Maps to:** FR-010, FR-011
- **Depends on:** TASK-001, TASK-002
- **Estimated effort:** M
- **Status:** 🔴 Blocked

---

## Phase 3: Secondary Features

[Same format]

---

## Phase 4: Polish and Edge Cases

[Same format — includes a11y, responsive, error handling, empty states]

---

## Dependency Graph (textual)

```
TASK-001 (Foundation)
├── TASK-002 (Auth setup)
│   ├── TASK-010 (User registration flow)
│   └── TASK-011 (User login flow)
├── TASK-003 (DB schema)
│   ├── TASK-010
│   └── TASK-012 (Dashboard page)
└── TASK-004 (Design tokens)
    └── TASK-010
```

## Summary

| Phase | Tasks | Backend | Frontend | Design | QA | Estimated total |
|-------|-------|---------|----------|--------|----|----------------|
| Foundation | [N] | [N] | [N] | [N] | [N] | [N days] |
| Core journeys | [N] | [N] | [N] | [N] | [N] | [N days] |
| Secondary | [N] | [N] | [N] | [N] | [N] | [N days] |
| Polish | [N] | [N] | [N] | [N] | [N] | [N days] |
| **Total** | **[N]** | **[N]** | **[N]** | **[N]** | **[N]** | **[N days]** |

---

## Backend tasks

See `.backend/<feature>/BACKEND_TASKS.md` for detail.

- **Count:** [N]
- **Effort breakdown:** Small: X | Medium: Y | Large: Z
- **Critical path:** [brief description]

## Frontend tasks

See `.frontend/<feature>/FRONTEND_TASKS.md` for detail.

- **Count:** [N]
- **Effort breakdown:** Small: X | Medium: Y | Large: Z
- **Critical path:** [brief description]

## Design tasks

See `.design/<feature>/DESIGN_TASKS.md` for detail.

- **Count:** [N]
- **Effort breakdown:** Small: X | Medium: Y | Large: Z
- **Critical path:** [brief description]

## QA tasks

See `.qa/<feature>/QA_TASKS.md` for detail.

- **Count:** [N]
- **Effort breakdown:** Small: X | Medium: Y | Large: Z
- **Critical path:** [brief description]
```

### Step 4.5: Cross-reference discipline task files

The TASKS.md template includes summary sections for each discipline (Backend / Frontend / Design / QA) that cross-reference the detailed discipline-specific task files produced by the sibling flows.

For each discipline, check whether its TASKS file exists and populate accordingly:

- **Backend:** read `.backend/<feature>/BACKEND_TASKS.md` if present; otherwise emit `Backend tasks: TBD — run \`backend-flow\` to generate.`
- **Frontend:** read `.frontend/<feature>/FRONTEND_TASKS.md` if present; otherwise emit `Frontend tasks: TBD — run \`frontend-flow\` to generate.`
- **Design:** read `.design/<feature>/DESIGN_TASKS.md` if present; otherwise emit `Design tasks: TBD — run \`design-flow\` to generate.`
- **QA:** read `.qa/<feature>/QA_TASKS.md` if present; otherwise emit `QA tasks: TBD — run \`qa-flow\` to generate.`

When the discipline file exists, populate the section with:
- Task count (from the discipline's task list)
- Effort breakdown (count of Small / Medium / Large tasks)
- Critical path (one-line description of the blocking sequence, if evident)

This keeps TASKS.md as the single-pane overview while each discipline owns its detailed decomposition.

### Step 5: Save

Save to `.pm/<feature-name>/TASKS.md`.

## Rules

- Every user story from the PRD must map to at least one task
- Every task must have testable acceptance criteria — "looks good" is not acceptance criteria
- Tasks should be 1-3 days of effort. If bigger, split them.
- Foundation tasks are always Phase 1 — never skip the scaffold
- Include design tasks in the flow — "design the X screen" is a real task with real deliverables
- Include testing in every task's acceptance criteria, not as a separate "write tests" task
- The dependency graph must be acyclic — if you create a circular dependency, something is wrong
- Effort estimates use T-shirt sizes: S (half day), M (1-2 days), L (3+ days — should probably be split)
- Every task's status starts at 🟡 (Ready) or 🔴 (Blocked) — nothing starts in progress
