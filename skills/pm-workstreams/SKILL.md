---
name: pm-workstreams
description: Reads prior PM artifacts from `.pm/<feature>/PRD.md` and `ARCHITECTURE.md` and produces `.pm/<feature>/WORKSTREAMS.md` — the discipline breakdown (backend / frontend / design / QA) with ownership boundaries and interfaces. Use when user says "split into workstreams", "divide the work", "who does what", or when the pm-flow orchestrator triggers this step. Not for generic team topology discussions or org-chart planning.
---

# PM Workstreams — Discipline Breakdown

Take the PRD and architecture and split them into clear, parallel workstreams. Each workstream gets its own scope, responsibilities, dependencies on the others, and definition of done.

## How to run

### Step 1: Gather inputs

Read these in order:
1. `.pm/<feature-name>/PRD.md` — user stories and functional requirements
2. `.pm/<feature-name>/ARCHITECTURE.md` — technical decisions and system design
3. `.pm/<feature-name>/GRILL_SUMMARY.md` — constraints and team info
4. Explore the codebase for existing patterns in each discipline

### Step 2: Ask workstream-specific questions

- "Does your team have dedicated backend, frontend, and design roles, or are people full-stack?"
- "Is there a dedicated QA owner, or do devs own their own testing (full-stack pattern)?"
- "Is there a designer involved, or should the design workstream be developer-handled?"
- "Are there any existing design systems or component libraries to use?"
- "What's the handoff process between design and development in your team?"

Adapt the workstreams based on the answers. A solo founder needs different splits than a team of 10.

### Step 3: Define workstream boundaries

For each workstream, define:

1. **What it owns** — the specific outputs this workstream is responsible for
2. **What it consumes** — inputs it needs from other workstreams
3. **What it produces** — outputs other workstreams depend on
4. **Interface contracts** — the specific agreements between workstreams

### Step 4: Write the workstreams document

```markdown
# Workstreams: [Feature/Project Name]

**Based on:** PRD v[X], Architecture v[X]
**Date:** [Today]

---

## Workstream Overview

| Workstream | Owner(s) | Description | Depends on |
|-----------|----------|-------------|------------|
| Backend | [TBD] | [1-line scope] | Design (API contracts) |
| Frontend | [TBD] | [1-line scope] | Backend (APIs), Design (specs) |
| Design | [TBD] | [1-line scope] | PRD (requirements) |
| QA | [TBD] | [1-line scope] | PRD (FR-XXX), Architecture (test seams), Design (a11y) |

---

## 1. Design Workstream

### Scope
[What the design workstream is responsible for]

### Deliverables
- [ ] User flow diagrams for each key journey
- [ ] Wireframes for each page/screen
- [ ] Visual design for each page/screen (if applicable)
- [ ] Component specifications (spacing, typography, colors)
- [ ] Empty states, error states, loading states
- [ ] Responsive breakpoints defined
- [ ] Design tokens / style guide (if no existing design system)

### Functional requirements covered
[List the FR-XXX IDs this workstream addresses]

### Dependencies
- **Needs from PRD:** Finalized user stories and edge cases
- **Produces for Frontend:** Design specs, component inventory, interaction patterns
- **Produces for Backend:** Form field specifications, validation rules, data display requirements

### Definition of done
- All user journeys from the PRD have corresponding designs
- Edge cases and error states are designed
- Design is reviewed and approved by stakeholders
- Specs are detailed enough for a developer to implement without ambiguity

---

## 2. Backend Workstream

### Scope
[What the backend workstream is responsible for]

### Deliverables
- [ ] Database migrations and schema
- [ ] API endpoints (per ARCHITECTURE.md)
- [ ] Authentication and authorization logic
- [ ] Business logic and validation
- [ ] External service integrations
- [ ] Background jobs / async processing (if applicable)
- [ ] API documentation (OpenAPI / Swagger)
- [ ] Unit and integration tests

### Functional requirements covered
[List the FR-XXX IDs this workstream addresses]

### Dependencies
- **Needs from Design:** Data requirements, validation rules
- **Needs from Architecture:** API contracts, data model, stack decisions
- **Produces for Frontend:** Working API endpoints with documentation
- **Produces for Frontend:** Auth tokens / session management

### API contracts (interface with Frontend)

| Endpoint | Request shape | Response shape | Status |
|----------|--------------|----------------|--------|
| POST /api/v1/[resource] | `{ field: type }` | `{ id, field, ... }` | Not started |
| GET /api/v1/[resource] | query params | `{ data: [...], meta: {...} }` | Not started |

### Definition of done
- All API endpoints functional and documented
- Data model migrated and seeded
- Auth flow working end-to-end
- Tests passing with >80% coverage on business logic
- API documentation up to date

---

## 3. Frontend Workstream

### Scope
[What the frontend workstream is responsible for]

### Deliverables
- [ ] Pages / routes (per ARCHITECTURE.md)
- [ ] Components (matching design specs)
- [ ] API integration (consuming backend endpoints)
- [ ] State management
- [ ] Form validation (client-side)
- [ ] Loading, empty, and error states
- [ ] Responsive layout
- [ ] Accessibility (keyboard nav, screen readers, ARIA)
- [ ] Unit and integration tests

### Functional requirements covered
[List the FR-XXX IDs this workstream addresses]

### Dependencies
- **Needs from Design:** Component specs, interaction patterns, visual assets
- **Needs from Backend:** Working API endpoints, API documentation, auth flow
- **Produces for Users:** The actual interface

### Definition of done
- All pages functional and matching design specs
- API integration complete and error-handling in place
- Responsive across defined breakpoints
- Accessibility audit passing
- Tests passing on critical user flows

---

## 4. QA Workstream

### Scope
Test strategy, automation, and review of the build.

### Deliverables
- [ ] `QA_STRATEGY.md` — test approach aligned to `testingRigor`
- [ ] `TEST_CASES.md` — cases mapped to FR-XXX and journeys
- [ ] `QA_TASKS.md` — automation and manual task breakdown
- [ ] `QA_REVIEW.md` — green/red status per release
- [ ] API contract tests (block merge when `testingRigor = full`)
- [ ] E2E coverage for critical journeys
- [ ] Accessibility checks against Design component specs
- [ ] Regression gap log

### Functional requirements covered
[List the FR-XXX IDs this workstream addresses — every `must` FR must be covered]

### Dependencies
- **Needs from PRD:** FR-XXX and user journeys
- **Needs from Architecture:** Test seams, `testingRigor` level
- **Needs from Design:** Component specs for a11y
- **Needs from Backend:** API contracts for integration tests
- **Needs from Frontend:** Routes + UI for E2E; E2E selector stability
- **Produces for PM:** Green-for-release definition and status

### Interface contracts
- **Backend:** API contract tests block merge when `testingRigor = full`
- **Frontend:** E2E selector stability — frontend commits to not breaking test selectors without coordination
- **PM:** Green-for-release definition

### Definition of done
- qa-review produces green status
- All `must` FR-XXX covered
- Regression gaps logged and prioritized

---

## Cross-Workstream Agreements

### Communication
[How workstreams coordinate: standups, async updates, shared channel]

### Handoff protocol
1. Design → Frontend: [How designs are delivered — Figma, specs, tokens]
2. Backend → Frontend: [How API readiness is communicated — docs, mock servers]
3. Frontend → QA: [How features are submitted for testing]

### Conflict resolution
[What happens when a design requirement conflicts with a technical constraint]

### Parallel work strategy
[What can start in parallel vs what's sequential]

```
Design:   ████████░░░░░░░░░░░░
Backend:  ░░░░████████████░░░░
Frontend: ░░░░░░░░████████████
```

[Explanation of overlap and parallel tracks]
```

### Step 5: Save

Save to `.pm/<feature-name>/WORKSTREAMS.md`.

## Rules

- Design workstream comes FIRST in the sequence — it unblocks both backend and frontend
- Every functional requirement (FR-XXX) from the PRD must appear in at least one workstream
- Interface contracts between workstreams must be specific enough to work from — not "the API"
- If there's no designer, the design workstream still exists — it just becomes wireframes and component specs done by the developer
- Flag any requirements that span multiple workstreams and need coordination
- The parallel work strategy should be realistic — don't pretend everything can be done at once if there are hard dependencies
