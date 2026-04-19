---
name: qa-brief-intake
description: Reads `.qa/<feature>/QA_BRIEF.md` (from pm-handoff or fallback to PRD + ARCHITECTURE), audits the existing test infrastructure in the codebase, classifies each QA concern as clear/ambiguous/missing, runs an embedded gap-fill grill on the ambiguous and missing items, and produces `.qa/<feature>/QA_INTAKE.md`. Use when qa-flow invokes it or when user says "intake the qa brief", "audit the test setup". Not for generic QA consulting or for replanning PM-level scope.
---

# QA Brief Intake — Brief + Codebase Test Reality + Gap-Fill Grill

Turns an incoming QA brief into an actionable starting point for `qa-strategy`.

## How to run

### Step 1: Load the brief

Read `.qa/<feature>/QA_BRIEF.md`. If absent, try `.pm/<feature>/PRD.md` plus `.pm/<feature>/ARCHITECTURE.md`. If neither, abort and ask the user to paste one.

Extract:
- Feature name and purpose
- Risk surface (user-facing flows, data integrity, auth, payments)
- Acceptance criteria hinted or declared
- Performance / a11y / compliance constraints
- Known test environments or staging targets

### Step 2: Audit the codebase

Read:
- **Node.js:** `package.json` test-related deps (`vitest`, `jest`, `playwright`, `cypress`, `@testing-library/*`, `@playwright/*`, `testcontainers`, `msw`)
- **Python:** `pyproject.toml` / `requirements.txt` (`pytest`, `pytest-cov`, `tox`, `hypothesis`)
- **Go:** `go.mod` (`testing` stdlib usage, `testify`, `gomock`)
- **CI config:** `.github/workflows/*`, `.gitlab-ci.yml`, `.circleci/config.yml`
- **Coverage config:** `vitest.config.ts` coverage block, `jest.config.js` `coverageThreshold`, `.coveragerc`, `codecov.yml`
- **Fixture directories:** `__fixtures__/`, `tests/fixtures/`, `seeds/`, `factories/`
- Existing test file counts by type (unit / integration / E2E, inferred from filename patterns like `*.test.*`, `*.spec.*`, `*.e2e.*`, `tests/integration/`)

### Step 3: Classify every QA concern

For each of these, mark ✅ clear / ⚠️ ambiguous / ❌ missing:

- Test framework (unit)
- Test framework (integration)
- Test framework (E2E)
- Test runner / CI
- Coverage tool
- Coverage target
- Test data / fixtures strategy
- Mocking strategy
- A11y testing tooling
- Perf testing tooling
- Visual regression tooling
- Bug tracking integration

### Step 4: Run gap-fill grill

Ask one question at a time, ordered Critical → Important, **only for items marked ⚠️ or ❌**. Never re-ask ✅ items.

For each question, offer a default the user can accept with "yes":

- "Unit framework — Vitest or Jest? Default: Vitest for Vite projects, Jest otherwise."
- "E2E — Playwright or Cypress? Default: Playwright unless team already owns Cypress."
- "Coverage target? Default: match `testingRigor` from PROJECT_PROFILE.md (mvp=reasonable non-numeric; full=80% lines / 70% branches)."
- "Mocking strategy — MSW, in-memory stubs, or record/replay? Default: MSW for HTTP, manual stubs for units."
- "A11y tooling — axe-core + jest-axe, or manual only? Default: axe in CI for full; manual for mvp."
- "Bug tracker — GitHub Issues / Jira / Linear? Default: whatever the repo already integrates with."

If the user answers vaguely, probe with "concretely, which?" — don't accept "we'll figure it out later" for Critical items.

### Step 5: Write QA_INTAKE.md

Save to `.qa/<feature>/QA_INTAKE.md`:

```markdown
# QA Intake: [Feature/Project Name]

**Date:** [today]
**Brief:** QA_BRIEF.md (or PRD.md + ARCHITECTURE.md fallback)
**Codebase:** [existing / greenfield]

## Testing context summary

[2-3 sentences: what the feature is, what risk surface it exposes, what testing posture the codebase currently has.]

## Codebase audit

**What exists:**
- Unit framework detected: [vitest 1.x / jest 29 / none]
- Integration setup: [testcontainers / none]
- E2E framework detected: [playwright / cypress / none]
- CI: [github actions / gitlab / circleci / none]
- Coverage tool: [c8 / istanbul / none], current threshold: [X% / not enforced]
- Fixtures: [`tests/fixtures/`, N files / none]
- Existing test counts: unit [N], integration [N], E2E [N]

**What's missing:**
- [e.g. no a11y tooling, no visual regression, no perf harness]

## Classification

| Concern | Status | Reason |
|---------|--------|--------|
| Test framework (unit) | ✅ clear | vitest already configured |
| Test framework (integration) | ⚠️ ambiguous | vitest present but no integration dir |
| Test framework (E2E) | ❌ missing | no playwright/cypress deps |
| Test runner / CI | ✅ clear | `.github/workflows/ci.yml` runs tests |
| Coverage tool | ✅ clear | c8 via vitest |
| Coverage target | ⚠️ ambiguous | config has no threshold |
| Test data / fixtures | ❌ missing | no factories, no seeds |
| Mocking strategy | ⚠️ ambiguous | ad-hoc `vi.mock` scattered |
| A11y testing | ❌ missing | no axe |
| Perf testing | ❌ missing | none |
| Visual regression | ❌ missing | none |
| Bug tracking | ✅ clear | repo uses GitHub Issues |

## Grill resolutions

| Concern | Options considered | Chosen | Rationale |
|---------|-------------------|--------|-----------|
| Coverage target | none / 70% / 85% | 70% | matches `testingRigor = standard` in PROJECT_PROFILE.md |
| E2E framework | playwright / cypress | playwright | better TS story, already used on sister repo |

## Open questions

- [Anything the grill couldn't resolve — with a note on why]

## Ready-for-qa-strategy assessment

**[Low / Medium / High]** — [one sentence: what's still blocking, or why this is ready to hand off]
```

### Step 6: Hand off

Tell the user: "Intake done. Next: `qa-strategy` will use this + `testingRigor` from PROJECT_PROFILE.md to produce the test strategy."

## Rules

- Don't re-audit on every run; respect what exists and never re-ask decisions already marked ✅.
- If coverage is already at 80% but `testingRigor = mvp`, flag as "over-engineered for current profile" — non-blocking note, not a demand to tear it down.
- Never guess a testing tool — audit the codebase first; only propose defaults after the audit proves a gap.
- Never decide the strategy here. Framework trade-offs, test-pyramid shape, and per-layer targets belong to `qa-strategy`.
- Output is markdown only — never write to the user's source tree, never install deps, never scaffold test files.
