# QA Flow — Design Doc

**Date:** 2026-04-19
**Status:** Approved, ready for implementation planning
**Scope:** Add a dedicated QA / testing workstream to the skills pack — a full `qa-flow` orchestrator with 6 new skills, plus a cross-cutting `testingRigor` project profile field that 3 skills (pm-architecture, qa-strategy, qa-review) read to adjust behavior between `mvp` and `full` rigor.

---

## Motivation

Today the pack covers PM, design, backend, frontend, and kanban — but testing has no home. Each discipline's `*-tasks` skill mentions a test plan inline (Vitest, Playwright), and each `*-review` screenshots the finished build. Nobody plans the **test strategy** (pyramid, tools by level, coverage targets, risk matrix for a11y/perf/sec, test data), and nobody translates PRD FR-XXX items into **structured test cases** that back-end and front-end can share as a single source of truth.

The gap gets worse at the MVP end: devs write their own ad-hoc tests, coverage drifts, E2E selectors rot silently, and regression matrices exist in someone's head.

Fix: a `qa-flow` that mirrors the backend/frontend pattern (brief-intake → stack-like decisions → content → tasks → review), with a `testingRigor` profile field that collapses the skill for MVP projects instead of forcing the full ceremony.

---

## Decision 1 — `qa-flow` skill list

Six new skills, symmetric to `backend-flow` and `frontend-flow`.

| Skill | Role | Produces | Runs when |
|---|---|---|---|
| `qa-flow` | Orchestrator | (invokes the 5 below in order) | entry point |
| `qa-brief-intake` | Enrich brief with codebase audit | `.qa/<feature>/QA_INTAKE.md` | after `pm-handoff` (which wrote `QA_BRIEF.md`) |
| `qa-strategy` | Test pyramid + tools + coverage + risks + test data | `.qa/<feature>/QA_STRATEGY.md` | after intake |
| `qa-cases` | Convert PRD FR-XXX + journeys → Gherkin / Given-When-Then | `.qa/<feature>/TEST_CASES.md` | after strategy |
| `qa-tasks` | Vertical slices (per slice: case set + automation + fixtures + data) | `.qa/<feature>/QA_TASKS.md` | after cases |
| `qa-review` | Post-build execution report + failure triage + regression gaps | `.qa/<feature>/QA_REVIEW.md` | after build |

**Single path (no A/B split like design-flow):** QA always starts from `pm-handoff` — there's no "already have a QA brief from somewhere else" case. Matches `backend-flow`.

**Reads from** (via the intake and strategy steps):
- `.pm/<feature>/PRD.md` — FR-XXX + user journeys
- `.pm/<feature>/ARCHITECTURE.md` — what needs test seams
- `.pm/<feature>/PROJECT_PROFILE.md` — `testingRigor`
- `.design/<feature>/COMPONENT_SPECS.md` — a11y requirements per component (when present)
- `.backend/<feature>/BACKEND_API.md` — API contracts to test
- `.frontend/<feature>/FRONTEND_ROUTES.md` — routes to E2E
- Codebase — existing test files, CI config, coverage reports

**Does not duplicate dev-level tests.** `frontend-tasks` and `backend-tasks` already prescribe unit+integration testing inside the discipline (Vitest + RTL for frontend, whatever fits backend). `qa-strategy` specifies the pyramid and tool choices — dev tasks execute unit+integration per their own plans. `qa-*` owns:
- End-to-end tests (cross-discipline)
- Formal acceptance criteria from PRD
- Regression suite maintenance
- Post-build test execution and triage

---

## Decision 2 — `testingRigor` branching

Two values, matching `designMode`'s binary pattern:

- **`mvp`** — smoke pyramid only:
  - `qa-strategy`: unit where critical (risk-weighted), 1 E2E happy path per user journey, integration tests optional.
  - `qa-cases`: covers only FR-XXX items marked "must" in the PRD.
  - `qa-review`: skips visual regression, formal a11y matrix, and perf budgets. Smoke E2E pass/fail + manual spot-check report.
- **`full`** — complete pyramid:
  - `qa-strategy`: full pyramid (unit for every module, integration per boundary, E2E per journey incl. edge cases).
  - `qa-cases`: every FR-XXX mapped, edge cases specified, negative cases included.
  - `qa-review`: visual regression (screenshot diffs), a11y matrix (WCAG AA per component), perf budget (LCP/INP/CLS targets per route).

**`testingRigor` does not affect** `qa-flow`, `qa-brief-intake`, `qa-cases` structure, or `qa-tasks` structure. It changes *what content* `qa-strategy` and `qa-review` produce, not the shape of the artifacts.

---

## Decision 3 — Integration with existing skills

Eight existing skills get edits. `design-flow` and the design-* sub-skills are unchanged — design isn't affected by `testingRigor` directly (QA consumes design outputs in `qa-cases`, but design-flow doesn't need to know).

| Skill | Change |
|---|---|
| `pm-grill` | New "Testing profile" branch in Technical branches. Writes `testingRigor` to `PROJECT_PROFILE.md`. Extends the "Project Profile (both modes)" section with the new field + adds `qa-strategy`/`qa-review`/`pm-architecture` to the list of downstream readers. |
| `pm-intake` | Detect test infra mentions (Jest/Vitest/Playwright/Cypress/coverage targets) as `✅ Clear`; absence → `⚠️ Ambiguous` gap (important, not critical — resolvable in qa-strategy). |
| `pm-architecture` | New "Testability" subsection referencing `testingRigor`. `full` rigor implies seams for mocking, test database, contract-test-friendly API design, etc. `mvp` implies none of that is required upfront. |
| `pm-workstreams` | Add **QA** as a 4th workstream. Dependencies: depends on PRD + architecture; runs in parallel with backend/frontend/design. Hands back to build via `qa-tasks`. |
| `pm-tasks` | Add a QA section to `TASKS.md` that cross-references `QA_TASKS.md` for detail. Same pattern as how design/backend/frontend sections cross-reference their own `*_TASKS.md`. |
| `pm-handoff` | Generate `.qa/<feature>/QA_BRIEF.md` as a 4th brief. Stamp `designMode` + `testingRigor` in the brief header. Add a "QA" option to the workstreams-to-start menu. Fallback prompt for legacy projects asks **both** `designMode` and `testingRigor` when `PROJECT_PROFILE.md` is missing. |
| `kanban-sync` | Add `QA_TASKS.md` as a 4th source. Label `qa`. Hierarchy: Epic → Tasks with `Blocks` links, same as design/backend/frontend. |
| `frontend-stack`, `backend-stack` | Read `QA_STRATEGY.md` if present. **Warn** (not abort) in the decision log when the stack's chosen test tooling conflicts with `qa-strategy` (e.g., frontend-stack picks Cypress but qa-strategy specified Playwright). Testing-tool conflicts are negotiable — unlike shadcn, which is structural. |

**Fallback-prompt design for legacy projects (unified):**

When any of `pm-handoff`, `design-flow`, `frontend-stack` finds a missing or partial `PROJECT_PROFILE.md`, one canonical prompt:

> "Este projeto não tem PROJECT_PROFILE.md completo.
> Modo de design: **shadcn-theme** ou **custom-system**?
> Rigor de teste: **mvp** ou **full**?
> (Respostas gravam em `.pm/<feature-name>/PROJECT_PROFILE.md` — afetam design-tokens, design-components, qa-strategy, qa-review.)"

If the file exists but `testingRigor` is missing (existing projects pre-dating this change), ask only for `testingRigor`, preserve `designMode`. Symmetric for the reverse.

---

## Decision 4 — Artifacts

**New directory `.qa/<feature>/`:**

```
.qa/<feature>/
  ├── QA_BRIEF.md        (pm-handoff)
  ├── QA_INTAKE.md       (qa-brief-intake)
  ├── QA_STRATEGY.md     (qa-strategy)
  ├── TEST_CASES.md      (qa-cases)
  ├── QA_TASKS.md        (qa-tasks)
  └── QA_REVIEW.md       (qa-review)
```

**`PROJECT_PROFILE.md` schema extension:**

```markdown
# Project Profile
designMode: shadcn-theme | custom-system
uiFramework: shadcn/ui | <other> | none
testingRigor: mvp | full
```

One line added. All seven existing readers keep working (they read only the fields they care about); three new readers pick up the new field (`pm-architecture`, `qa-strategy`, `qa-review`).

---

## Decision 5 — Execution ordering

After `pm-handoff`, the recommended parallel strategy:

```
pm-handoff
  ├──► design-flow      ──┐
  ├──► backend-flow     ──┤
  └──► qa-flow          ──┤   (parallel: each reads PM artifacts)
                          │
  ┌─── frontend-flow  ◄───┘   (depends on design specs + backend API)
  │
  └──► build (all disciplines)
         │
         ├──► design-review
         ├──► backend-review
         ├──► frontend-review
         └──► qa-review    (parallel post-build)
```

`pm-workstreams` documents this strategy — no code enforcement, but the workstream graph makes the dependencies explicit.

---

## Decision 6 — Rollout and migration

**New projects:** `pm-grill` asks `testingRigor` in the new Testing profile branch. No silent default.

**Existing projects with partial `PROJECT_PROFILE.md`:** one of the three fallback points (`pm-handoff`, `design-flow`, `frontend-stack`) asks the missing fields and rewrites the file. Uses the unified prompt from Decision 3.

**Existing `.qa/<feature>/` artifacts don't exist yet** (this is a new feature), so no legacy artifact migration.

**Conflict between `testingRigor: mvp` and existing over-engineered test suites:** not our problem — `qa-strategy` in `mvp` mode just documents the smoke approach going forward. Existing coverage stays until the team decides to prune.

**README + `docs/skills-flow.md`:** updated in a final commit after the 14 skill edits land.

---

## Summary of files changed

| File | Change |
|---|---|
| `skills/qa-flow/SKILL.md` | **New** — orchestrator |
| `skills/qa-brief-intake/SKILL.md` | **New** — codebase audit |
| `skills/qa-strategy/SKILL.md` | **New** — pyramid + tools + coverage + risks; testingRigor branching |
| `skills/qa-cases/SKILL.md` | **New** — FR-XXX → Gherkin |
| `skills/qa-tasks/SKILL.md` | **New** — vertical slices |
| `skills/qa-review/SKILL.md` | **New** — post-build execution report; testingRigor branching |
| `skills/pm-grill/SKILL.md` | Edit — Testing profile branch + Project Profile section extension |
| `skills/pm-intake/SKILL.md` | Edit — test infra detection row |
| `skills/pm-architecture/SKILL.md` | Edit — Testability subsection referencing testingRigor |
| `skills/pm-workstreams/SKILL.md` | Edit — add QA as 4th workstream |
| `skills/pm-tasks/SKILL.md` | Edit — add QA section to TASKS.md template |
| `skills/pm-handoff/SKILL.md` | Edit — generate QA_BRIEF; unified fallback prompt |
| `skills/kanban-sync/SKILL.md` | Edit — ingest QA_TASKS.md |
| `skills/frontend-stack/SKILL.md` | Edit — read QA_STRATEGY for warn-conflict |
| `skills/backend-stack/SKILL.md` | Edit — same |
| `README.md` | Edit — add QA to flow diagram; Project modes section expanded with testingRigor; add `.qa/` artifacts section |
| `docs/skills-flow.md` | Edit — new QA flow mermaid; update top-level + PROJECT_PROFILE maps |
| `docs/plans/2026-04-19-qa-flow-design.md` | This doc |

**New runtime artifacts:** `.qa/<feature>/*.md` (created by consumer projects, not this repo).

---

## Open questions (deferred, not blocking v1)

- **`testingRigor: standard` (middle value)** — some projects are neither MVP nor full. Defer until a real case shows up.
- **Security-specific QA flow** — penetration testing, threat modeling. Out of scope for v1.
- **Performance QA as its own flow** — covered as a sub-matrix in `qa-review` `full` mode. Separate flow only if a project needs continuous perf tracking.
- **Cross-project regression library** — shared Gherkin scenarios across features. Not in v1.
- **Integration with existing superpowers:test-driven-development skill** — that skill is per-task TDD discipline; qa-flow is strategic. Complementary, no wiring needed.
