---
name: qa-tasks
description: Reads `.qa/<feature>/QA_STRATEGY.md` and `TEST_CASES.md` and produces `.qa/<feature>/QA_TASKS.md` — vertical-sliced QA implementation tasks (each slice = case group + automation + fixtures + CI wiring). Task shape matches backend-tasks and frontend-tasks so kanban-sync ingests it with no translation. Use when qa-flow invokes it or when user says "decompose the qa work", "plan the test automation". Not for writing actual test code (that's execution, outside this pack) or for defining the strategy (that's qa-strategy).
---

# QA Tasks — Vertical-Sliced QA Implementation Plan

Turns the QA strategy and test case inventory into a sequenced, kanban-sync-compatible work list.

## How to run

### Step 1: Load inputs

Read:
- `.qa/<feature>/QA_STRATEGY.md`
- `.qa/<feature>/TEST_CASES.md`
- `.qa/<feature>/QA_INTAKE.md`

Optional (for wiring context — warn but proceed if missing):
- `.backend/<feature>/BACKEND_API.md` (endpoint contracts for API tests)
- `.frontend/<feature>/FRONTEND_ROUTES.md` (routes for E2E tests)
- `.backend/<feature>/BACKEND_TASKS.md` / `.frontend/<feature>/FRONTEND_TASKS.md` (for cross-discipline dependency IDs)

Abort if QA_STRATEGY or TEST_CASES are missing.

### Step 2: Decompose test cases into vertical slices

A QA vertical slice = one shippable automation increment that covers a coherent case group end-to-end. Every slice contains:

- **Case group** — e.g., all login journey cases, all API CRUD cases for a single resource, all empty-state cases for a route
- **Test automation code** — the specs/specs files that exercise the cases
- **Fixtures / seed data** — factories, DB seeds, mocks, or recorded responses the cases need
- **CI wiring** — pipeline job or step that runs the slice
- **Doc updates** — mark covered cases as `automated` in TEST_CASES.md; update QA_STRATEGY.md if scope shifted

Never split "write tests" and "wire CI" into separate tasks for the same case group. A slice is done only when its cases run green in CI.

### Step 3: Size each slice

Use one of these exact values (kanban-sync contract — must match backend-tasks / frontend-tasks literal strings):

- `Small` — 1–4 hours. A single case group, no new infra.
- `Medium` — half day. A case group that also touches fixtures or a small CI change.
- `Large` — 1–2 days. Case group plus new harness, new CI job, or new fixture pipeline.

Anything > Large gets split until it fits.

### Step 4: Order by risk and dependency

1. **Critical-path cases first** — happy-path journeys and smoke cases that gate release.
2. **Shared test infra before specific cases** — harness setup, fixture factories, CI wiring scaffold ship before the slices that depend on them.
3. **Fixtures before dependent cases** — seed data and mocks land with (or before) the cases that consume them.
4. **High-risk areas early** — areas flagged in QA_STRATEGY risk register get automation before low-risk polish.
5. **Edge cases and negative paths** after their happy-path sibling is green.

### Step 5: Flag cross-discipline dependencies

E2E automation requires running backend endpoints and rendered frontend routes. API contract tests require deployed endpoints. Flag every upstream dependency explicitly:

- `Blocked by: BACKEND_TASK_006` — endpoint must be live before the API slice can run
- `Blocked by: FRONTEND_TASK_005` — route must render before the E2E slice can run
- `Blocked by: DESIGN_TASK_004` — only if visual-regression cases depend on design landing

Do not work around a missing dependency. A blocked QA slice stays blocked and visible in the board.

### Step 6: Write QA_TASKS.md

Save to `.qa/<feature>/QA_TASKS.md`:

````markdown
# QA Tasks: [Feature Name]

**Strategy:** .qa/<feature>/QA_STRATEGY.md
**Test cases:** .qa/<feature>/TEST_CASES.md
**Backend API:** .backend/<feature>/BACKEND_API.md
**Frontend routes:** .frontend/<feature>/FRONTEND_ROUTES.md

## Summary

| Phase | Count | Effort |
|-------|-------|--------|
| Harness + fixtures | 3 | ~1 day |
| Critical-path automation | 5 | ~3 days |
| Secondary coverage | 4 | ~2 days |
| CI + reporting | 2 | ~1 day |

## Phase 1 — Harness + fixtures

### QA_TASK_001 — Playwright harness + base fixtures

**Description:** Stand up the Playwright project per QA_STRATEGY.md §Tooling. Create shared `auth.fixture.ts` (logged-in user context) and `users.fixture.ts` (factory for test user rows).
**Effort:** Medium
**Blocks:** QA_TASK_002, QA_TASK_003, QA_TASK_004
**Blocked by:** BACKEND_TASK_001 (server runnable), FRONTEND_TASK_001 (app runnable)
**Test cases covered:** infra only — no product cases
**Files to create:**
- `tests/playwright.config.ts`
- `tests/fixtures/auth.ts`
- `tests/fixtures/users.ts`
**CI integration:** new job `e2e` in `.github/workflows/ci.yml` (scaffold only, no specs yet)
**Done when:**
- [ ] `npx playwright test` runs with zero specs and exits 0
- [ ] `auth.fixture.ts` logs a seeded user in and exposes a ready page context
- [ ] `e2e` CI job wired and runs green on main

## Phase 2 — Critical-path automation

### QA_TASK_002 — Login journey E2E

**Description:** Automate the full login happy path + invalid-credentials + locked-account cases from TEST_CASES.md.
**Effort:** Small
**Blocks:** QA_TASK_005 (post-login flows)
**Blocked by:** QA_TASK_001, BACKEND_TASK_005 (login endpoint live), FRONTEND_TASK_004 (login route rendered)
**Test cases covered:** JOURNEY_LOGIN_HAPPY, JOURNEY_LOGIN_BAD_CREDS, JOURNEY_LOGIN_LOCKED
**Files to create:**
- `tests/e2e/login.spec.ts`
**CI integration:** runs in existing `e2e` job
**Done when:**
- [ ] Automation code runs green locally
- [ ] Added to CI pipeline in `e2e` job
- [ ] All covered cases marked as automated in TEST_CASES.md

### QA_TASK_003 — API users CRUD contract tests

**Description:** Contract tests for POST/GET/PATCH/DELETE `/v1/users` covering happy path, validation errors, auth errors.
**Effort:** Medium
**Blocks:** —
**Blocked by:** QA_TASK_001, BACKEND_TASK_006
**Test cases covered:** API_USERS_CREATE, API_USERS_READ, API_USERS_UPDATE, API_USERS_DELETE, API_USERS_401
**Files to create:**
- `tests/api/users.spec.ts`
- `tests/fixtures/user-factory.ts`
**CI integration:** new `api` job in `.github/workflows/ci.yml`
**Done when:**
- [ ] All 5 cases pass against a running backend
- [ ] `api` CI job added and green
- [ ] Covered cases marked as automated in TEST_CASES.md

## Phase 3 — Secondary coverage

### QA_TASK_006 — Empty states and error boundaries

**Description:** E2E coverage of empty-state and error-boundary cases across dashboard and project-detail routes.
**Effort:** Small
**Blocked by:** QA_TASK_001, FRONTEND_TASK_012
**Test cases covered:** UI_DASHBOARD_EMPTY, UI_PROJECT_404, UI_PROJECT_500
**Files to create:**
- `tests/e2e/empty-states.spec.ts`
**CI integration:** runs in `e2e` job
**Done when:**
- [ ] All 3 cases pass
- [ ] Marked automated in TEST_CASES.md

## Phase 4 — CI + reporting

### QA_TASK_009 — Test report artifact + PR comment

**Description:** Upload Playwright HTML report as CI artifact; post pass/fail summary as PR comment.
**Effort:** Small
**Blocked by:** QA_TASK_001
**Test cases covered:** infra only
**Files to create/modify:**
- `.github/workflows/ci.yml` (artifact upload + comment step)
**CI integration:** extends `e2e` job
**Done when:**
- [ ] Failing run shows downloadable HTML report on the PR
- [ ] PR comment summarises pass/fail counts

## Dependency graph

```
001 ─┬─ 002 ─ 005
     ├─ 003
     ├─ 006
     └─ 009
```

Cross-discipline dependencies:
- 002 blocked by BACKEND_TASK_005, FRONTEND_TASK_004
- 003 blocked by BACKEND_TASK_006
- 006 blocked by FRONTEND_TASK_012

## Parallelizable

- 002 ∥ 003 ∥ 006 (different case groups, independent fixtures)
````

### Step 7: Hand off

Tell the user: "Tasks decomposed. These feed kanban-sync (label: `qa`). qa-review runs post-build to verify."

## Rules

- Every task is a vertical slice — case group + automation + fixtures + CI wiring land together. Never ship a slice whose cases are not running in CI.
- Each task must fit in 1–2 days max. Anything larger gets split until it fits a `Small` / `Medium` / `Large` box.
- `Done when` acceptance criteria must be objectively verifiable — CI job green, cases marked `automated` in TEST_CASES.md, report artifact present. Never "tests feel comprehensive."
- No task reads "write some tests." Every task names the exact case IDs from TEST_CASES.md it covers.
- Effort values are literal `Small` / `Medium` / `Large` only — this is the kanban-sync contract shared with backend-tasks and frontend-tasks. Markdown only; this skill never writes real test code.
