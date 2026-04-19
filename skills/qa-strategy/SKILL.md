---
name: qa-strategy
description: Reads `.qa/<feature>/QA_INTAKE.md`, `.pm/<feature>/PRD.md`, and `.pm/<feature>/PROJECT_PROFILE.md` (for `testingRigor`), and produces `.qa/<feature>/QA_STRATEGY.md` — the canonical test strategy document (pyramid, tools per layer, coverage targets, risk matrix for a11y/perf/sec, test data strategy, CI gates). Branches behavior on `testingRigor`: `mvp` mode outputs a smoke pyramid; `full` mode outputs a complete pyramid with explicit coverage targets and a11y/perf/visual-regression matrices. Use when qa-flow invokes it or when user says "define the test strategy". Not for executing tests or for ad-hoc test debugging.
---

# QA Strategy — Test Pyramid, Tools, Coverage, Risks

Locks in the pyramid shape, tool per layer, coverage targets, risk matrices, test data strategy, and CI gates for a specific project.

## How to run

### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — get `testingRigor`. MUST exist (pm-handoff, design-flow, or qa-flow creates it). Branch behavior at Step 1.5 based on this value.
- `.qa/<feature>/QA_INTAKE.md` — the codebase audit + gap-fill grill output from `qa-brief-intake`. Tools already in use, CI config, coverage tooling, deferred decisions.
- `.pm/<feature>/PRD.md` — FR-XXX list + user journeys. These drive case scope in `qa-cases`; strategy only needs the shape (how many FRs, how many journeys, critical paths).
- `.pm/<feature>/ARCHITECTURE.md` — seams for mocking, data model, external integrations. The Testability section (if present) already states what `full` rigor needs.
- Optional: `.backend/<feature>/BACKEND_API.md` (endpoints to contract-test), `.frontend/<feature>/FRONTEND_ROUTES.md` (routes to E2E).

If `QA_INTAKE.md` does not exist, abort. Tell the user: "Run `qa-brief-intake` first — I need the codebase audit + gap-fill grill results."

### Step 1.5: Branch on testingRigor

- **`testingRigor: mvp`** → skip Steps 5, 6, 7 (the a11y / perf / security matrices that are full-only). Go to the **MVP mode** section at the end of this skill. Canonical output is a slim QA_STRATEGY.md with smoke pyramid + tool row + risk-weighted coverage note.
- **`testingRigor: full`** → follow Steps 2–10 as written (complete strategy with explicit numeric coverage targets and all three risk matrices).

### Step 2: Build the test pyramid

Declare ratios per layer and render a text-art pyramid in the artifact. `full` mode uses the classic shape (many unit, fewer integration, few E2E); `mvp` mode collapses to a smoke shape (few unit at risk points, 1 E2E per journey).

**Full mode pyramid (typical ratio):**

```
         /\
        /E2\         ~10 E2E (journeys + critical edges)
       /----\
      / INTG \       ~40 integration (per boundary)
     /--------\
    /   UNIT   \     ~200+ unit (per module / pure function)
   /____________\
```

**MVP mode pyramid (smoke shape):**

```
       /\
      /E2\           1 E2E per critical user journey
     /----\
    / UNIT \         unit at risk points only
   /________\        (integration deferred)
```

State each layer's scope in one line:
- **Unit** — pure functions, services, reducers, utilities. No I/O.
- **Integration** — HTTP → DB round-trip, component + hook + context, cross-module flows. No mocks for seams that are real.
- **E2E** — full user journey from UI click to DB state and back. Real app, real browser, real API.

### Step 3: Choose tools per layer

For each row, either confirm what `QA_INTAKE.md` found already in use, or decide now if deferred. Every choice needs a rationale tied to the intake — never guess, never pick by popularity alone.

| Concern | Typical options | Default |
|---------|-----------------|---------|
| Unit | Vitest / Jest / pytest / go test | Vitest (Node) / pytest (Python) |
| Integration | Vitest + Supertest / Jest + Supertest / pytest + httpx / Testcontainers | Vitest + Supertest (Node) |
| E2E | Playwright / Cypress / Selenium | Playwright |
| Coverage | c8 / v8 / nyc / coverage.py | c8 (Node) / coverage.py (Python) |
| CI | GitHub Actions / GitLab CI / CircleCI | GitHub Actions |

If a row's tool is already locked in the codebase (intake reported its presence in `package.json` / `pyproject.toml` / CI config), keep it. Do not propose a rewrite unless the intake resolved to migrate.

### Step 4: Declare coverage targets

**Full mode — specific numbers:**

| Layer | Metric | Target | Enforced in CI |
|-------|--------|--------|----------------|
| Unit | Line coverage | 80% | yes (fail merge) |
| Unit | Branch coverage | 70% | yes |
| Critical path | Statement coverage | 100% | yes |
| Integration | Per-endpoint smoke | 100% of routes | yes |
| E2E | Per-journey | 100% of journeys | no (runs on main nightly) |

Define "critical path" explicitly — list the FR-XXX IDs that must hit 100%.

**MVP mode — risk-weighted, non-numeric:**

> "Reasonable coverage at risk points; no numeric target in CI. Merge blockers: (1) E2E smoke suite green, (2) unit suite green where declared. No coverage threshold enforced."

### Step 5: Risk matrix — a11y (full mode only)

Declare what's tested and how:

- **Standard:** WCAG 2.1 AA.
- **Per component:** every interactive component (button, input, modal, nav) has an axe-core assertion in its integration test file.
- **Keyboard-only path:** one E2E per critical journey walks through with keyboard only (Tab / Enter / Esc / Arrow keys). Asserts focus visibility and focus trap correctness.
- **Screen reader smoke:** manual checklist per release (VoiceOver on Safari, NVDA on Firefox) for the top-3 journeys. Logged in `QA_REVIEW.md`.
- **Tooling:** `@axe-core/playwright` for automated a11y in E2E. `jest-axe` or `@axe-core/react` for component-level.

MVP mode skips this matrix entirely (flagged at Step 1.5).

### Step 6: Risk matrix — perf (full mode only)

Declare budgets per route and enforcement method:

- **Frontend Core Web Vitals per route** — LCP < 2.5s, INP < 200ms, CLS < 0.1 (measured against 75th percentile of real users; in CI measured against a lab run).
- **Tooling:** Lighthouse CI wired into GitHub Actions on PRs touching `src/app/**` or equivalent. Fail budget in `lighthouserc.js`.
- **API latency per endpoint** — p95 < 300ms for read endpoints, p95 < 500ms for write endpoints. Measured via load test in CI (k6 / artillery) or manual baseline per release.
- **Perf regression gate:** nightly run on `main`, diff against baseline, open issue if regression > 10%.

MVP mode skips this matrix entirely.

### Step 7: Risk matrix — security (full mode only)

Declare coverage of the attack surface:

- **Input validation tests** — every API endpoint has integration tests covering: missing required field, wrong type, injection payload (SQL / XSS), oversized input. Must return 400 with error envelope, never 500.
- **Auth boundary tests** — per protected route: unauthenticated request → 401; authenticated but unauthorized user → 403; correct role → 200. Covered in integration suite.
- **Dependency scan** — Snyk or Dependabot enabled on the repo. Any `high` or `critical` advisory blocks merge. Reviewed weekly.
- **Secrets scanning** — pre-commit hook + GitHub secret scanning; any detection blocks.

MVP mode skips this matrix entirely.

### Step 8: Test data strategy

Declare both modes — this applies to `mvp` and `full`:

- **Fixtures location:** `tests/fixtures/` (unit + integration) or `e2e/fixtures/` (Playwright). Shape: one file per entity type, factory functions preferred over static JSON.
- **Seed strategy:** integration tests use a per-test transaction rollback (preferred) or a truncate + reseed per spec. E2E tests use a seeded test tenant per run, torn down after.
- **Production-like data handling:** never pull live data with PII. If a realistic dataset is needed, use a scrubbed dump (emails masked, names faked, ids regenerated). Document the scrubber tool + schedule.
- **Test tenant isolation:** E2E runs against a dedicated environment (`test.` subdomain / isolated DB schema). Never against staging that humans also use. Never against prod.
- **Flakiness hygiene:** tests that rely on time use a frozen clock (fake timers / `freezegun`). Tests that rely on network use MSW / VCR / recorded fixtures — not live external calls.

### Step 9: CI integration

Declare which gates block merge and where each runs. Cross-references `backend-tasks` and `frontend-tasks` — those tasks add the CI steps; `qa-strategy` defines what those CI steps must enforce.

**Full mode gates:**

| Gate | Where | Blocks merge? | Notes |
|------|-------|---------------|-------|
| Lint + typecheck | Every PR | yes | Existing pattern |
| Unit tests | Every PR | yes | Coverage threshold enforced |
| Integration tests | Every PR | yes | Real DB via docker-compose |
| E2E smoke | Every PR | yes | 1–2 critical journeys only; fast |
| E2E full | Nightly on `main` | no | Full journey matrix; opens issue on failure |
| Coverage | Every PR | yes | Fails if below Step 4 targets |
| a11y (axe) | Every PR | yes | Zero-violations policy on new components |
| Lighthouse CI | PRs touching UI | yes (budget) | LCP / INP / CLS per Step 6 |
| Dependency scan | Every PR | yes | Blocks on `high`+ |

**MVP mode gates:**

| Gate | Where | Blocks merge? |
|------|-------|---------------|
| Lint + typecheck | Every PR | yes |
| Unit tests (where declared) | Every PR | yes |
| E2E smoke | Every PR | yes |

No coverage threshold. No a11y / perf / dep-scan gates.

The `backend-tasks` and `frontend-tasks` skills add CI workflow files that implement these gates; this skill defines what those CI files must enforce. If the gates change, both `qa-strategy` and the `*-tasks` artifacts update in lock step.

### Step 10: Write QA_STRATEGY.md

Save to `.qa/<feature>/QA_STRATEGY.md`. Structure matches the rigor mode.

**Full mode template:**

````markdown
# QA Strategy: [Feature Name]

**testingRigor:** full
**Intake:** QA_INTAKE.md
**PRD:** PRD.md
**Architecture:** ARCHITECTURE.md

## Pyramid

```text
         /\
        /E2\         ~10 E2E
       /----\
      / INTG \       ~40 integration
     /--------\
    /   UNIT   \     ~200+ unit
   /____________\
```

| Layer | Scope | Approximate count |
|-------|-------|-------------------|
| Unit | pure functions, services, utils | 200+ |
| Integration | HTTP → DB, cross-module | ~40 |
| E2E | full user journeys | ~10 |

## Tools per layer

| Concern | Choice | Version | Rationale |
|---------|--------|---------|-----------|
| Unit | Vitest | 1.x | Already in package.json; ESM-native |
| Integration | Vitest + Supertest | — | Same runner as unit; faster feedback |
| E2E | Playwright | 1.4x | Intake resolved: team XP + multi-browser |
| Coverage | c8 | 9.x | Vitest default; V8 native |
| CI | GitHub Actions | — | Repo already on GH |

## Coverage targets

| Layer | Metric | Target | Enforced |
|-------|--------|--------|----------|
| Unit | Line | 80% | yes |
| Unit | Branch | 70% | yes |
| Critical path (FR-001, FR-002, FR-007) | Statement | 100% | yes |
| Integration | Per-route smoke | 100% routes | yes |
| E2E | Per-journey | 100% journeys | nightly |

## Risk matrix — a11y
[Step 5 content]

## Risk matrix — perf
[Step 6 content]

## Risk matrix — security
[Step 7 content]

## Test data strategy
[Step 8 content]

## CI integration
[Step 9 full-mode gates table]

## Boundaries with dev tasks

- Unit + integration tests inside a discipline are owned by that discipline's devs (backend devs write backend unit; frontend devs write frontend unit). `qa-strategy` defines the pyramid shape and tool; `backend-tasks` / `frontend-tasks` execute.
- E2E, cross-discipline contract tests, visual regression, and formal a11y/perf/security matrices are owned by QA. `qa-tasks` decomposes these.
- If a QA concern appears in a dev task (e.g., "write unit tests for the auth service"), it lives in `backend-tasks` / `frontend-tasks`, not `QA_TASKS.md`.

## Decision log

| # | Decision | Options considered | Chosen | Rationale |
|---|----------|-------------------|--------|-----------|
| 1 | E2E tool | Playwright / Cypress | Playwright | multi-browser + better CI DX |
| ... | | | | |
````

**MVP mode template:** see **MVP mode** section at the end of this skill.

### Step 11: Hand off

Tell the user: "Strategy locked. Next: `qa-cases` converts PRD FR-XXX items and user journeys into structured Gherkin test cases that match this pyramid."

## Rules

- Read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1. `testingRigor` changes whether Steps 5–7 (a11y / perf / security matrices) render. Never fall through to `full` when the profile says `mvp` — the slim strategy is the intentional output for MVP, not a degraded one.
- Never execute tests in this skill. The output is markdown planning only — tests run in dev tasks' CI or via `qa-review` post-build.
- Never guess tools. Use whatever `QA_INTAKE.md` reports already in use; if a layer has nothing, pick the default from Step 3's table and justify against the intake's language / framework context.
- Keep boundaries with dev tasks explicit. Dev tests unit + integration inside their discipline (per `backend-tasks` / `frontend-tasks`). QA owns E2E, cross-discipline contract tests, and the formal risk matrices. If this skill tries to define a backend unit test, stop — that belongs to `backend-tasks`.
- Markdown only. No code generated into the source tree from this skill — only `.qa/<feature>/QA_STRATEGY.md`.

---

## MVP mode

Use when `PROJECT_PROFILE.md` has `testingRigor: mvp`. Smoke-only strategy — intentionally slim.

### What this mode includes

- Smoke pyramid: unit tests at risk points only (no module-by-module sweep), 1 E2E happy path per critical user journey, integration tests **deferred** (not part of the merge gate).
- Tool choices: whatever `QA_INTAKE.md` reports already in use. Defaults if nothing present: **Playwright** for E2E, **Vitest** (Node) or **pytest** (Python) for unit.
- Coverage: "reasonable, risk-weighted, non-numeric." No CI threshold. Merge blockers are limited to (1) E2E smoke green, (2) declared unit tests green.
- Test data strategy (Step 8) — applies to mvp mode too. Same rules about fixtures, seeds, PII.
- CI integration (Step 9) — MVP gates table only.

### What this mode skips

- Step 5 — a11y risk matrix (no WCAG AA enforcement, no axe-core in CI, no keyboard-only path audit, no screen-reader smoke).
- Step 6 — perf risk matrix (no LCP/INP/CLS budgets, no Lighthouse CI, no API p95 gates).
- Step 7 — security risk matrix (no input-validation test sweep, no auth boundary battery, no dep-scan gate).
- Explicit numeric coverage targets in Step 4.
- Integration-layer mandate. Integration tests are allowed but not required; not a merge blocker.

These are deferred, not forbidden. When `testingRigor` moves to `full`, re-run this skill to expand the strategy.

### Write QA_STRATEGY.md (mvp shape)

Save to `.qa/<feature>/QA_STRATEGY.md`:

````markdown
# QA Strategy: [Feature Name] (mvp mode)

**testingRigor:** mvp
**Intake:** QA_INTAKE.md
**PRD:** PRD.md

## Pyramid (smoke shape)

```text
       /\
      /E2\           1 E2E per critical journey
     /----\
    / UNIT \         unit at risk points only
   /________\        (integration deferred)
```

## Tools per layer

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Unit | Vitest / pytest | Already in use per intake |
| E2E | Playwright | Multi-browser default |
| CI | GitHub Actions | Existing pattern |

(Integration / coverage tool / visual regression tool: deferred.)

## Coverage

Reasonable, risk-weighted, non-numeric. Merge blockers: E2E smoke green + declared unit suites green. No numeric coverage threshold in CI.

## Critical journeys (E2E scope)

- [Journey 1 from PRD] — 1 happy path
- [Journey 2 from PRD] — 1 happy path
- ...

## Test data strategy

[Step 8 content — same as full mode]

## CI integration

| Gate | Where | Blocks merge? |
|------|-------|---------------|
| Lint + typecheck | Every PR | yes |
| Unit tests (where declared) | Every PR | yes |
| E2E smoke | Every PR | yes |

## What this mode skips

- a11y matrix (WCAG AA per component, axe-core, keyboard-only path, screen-reader smoke)
- perf matrix (LCP/INP/CLS budgets, Lighthouse CI, API p95 gates)
- security matrix (input validation sweep, auth boundary battery, dep-scan gate)
- explicit numeric coverage targets
- integration test mandate

Deferred until `testingRigor` moves to `full`.

## Boundaries with dev tasks

- Unit tests at risk points: written by the discipline that owns the module (per `backend-tasks` / `frontend-tasks`).
- E2E smoke suite: owned by QA (per `qa-tasks`).
- No formal a11y / perf / security owner in mvp mode — devs apply judgment; QA re-evaluates if rigor rises.

## Decision log

| # | Decision | Chosen | Rationale |
|---|----------|--------|-----------|
| 1 | E2E tool | Playwright | intake already present |
| ... | | | |
````

### Hand off

Tell the user: "Strategy locked (mvp mode). Next: `qa-cases` converts the critical journeys and must-have FR-XXX items into Gherkin cases — it too will run in mvp mode and skip edge/negative case expansion."
