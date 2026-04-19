---
name: qa-review
description: Runs after the build is live. Reads `.qa/<feature>/QA_STRATEGY.md`, `TEST_CASES.md`, test execution output, and `.pm/<feature>/PROJECT_PROFILE.md` (for `testingRigor`). Produces `.qa/<feature>/QA_REVIEW.md` — pass/fail matrix per test case, triage of failures (test bug vs product bug vs infra issue), regression gaps, and follow-up tasks. In `mvp` mode outputs a smoke report; in `full` mode adds coverage report, visual regression diffs, a11y matrix, and perf budget compliance. Use when qa-flow invokes it post-build or when user says "qa review", "test the build". Not for running tests ad-hoc (execution is external) or for replacing design-review / frontend-review (those handle visual critique; this handles test execution results).
---

# QA Review — Post-Build Test Execution Report

Reports on test execution results against the declared strategy. Reads artifacts, does not run tests.

## How to run

### Step 1: Preconditions

Confirm:
1. A built app is running (preview URL, staging, or CI build under test).
2. Test execution output is available on disk or reachable: JUnit XML, Playwright HTML report, coverage JSON (lcov / cobertura), a11y scan results (axe JSON), visual regression snapshot diffs, perf traces (Lighthouse JSON).
3. `.qa/<feature>/QA_STRATEGY.md`, `TEST_CASES.md`, `QA_TASKS.md` exist.
4. `.pm/<feature>/PROJECT_PROFILE.md` exists.

If any of the above is missing, abort with a precise message:

> "Cannot run qa-review — missing [artifact]. Need a running build + recent test run output. Re-run tests via CI (or locally) and point me at the artifacts."

### Step 2: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — extract `testingRigor`.
- `.qa/<feature>/QA_STRATEGY.md` — pyramid, tools, coverage targets, risk matrix.
- `.qa/<feature>/TEST_CASES.md` — the canonical test case list (IDs, Gherkin).
- `.qa/<feature>/QA_TASKS.md` — task list with coverage references.
- Most recent test run artifacts (JUnit/Playwright/coverage/axe/Lighthouse). Record build SHA and run URL.

### Step 3: Branch on testingRigor

- `mvp`: produce a smoke report — pass/fail matrix + manual spot-check notes + critical-path regression checklist. Skip Steps 6–8 entirely.
- `full`: produce the complete report — everything `mvp` covers plus coverage report, visual regression diffs, a11y matrix, and perf budget compliance.

Announce the mode in the header of QA_REVIEW.md.

### Step 4: Build the pass/fail matrix

For each test case listed in `TEST_CASES.md`:

- Match case ID against execution output (test name / tag / file path).
- Columns: case ID, title, status (`pass` / `fail` / `skipped` / `missing`), duration, last run timestamp.
- Cases present in `TEST_CASES.md` but absent from execution output → mark `missing` (no automation yet).
- Cases present in execution output but not in `TEST_CASES.md` → note as "untracked" in a separate list (feeds Step 9 regression gaps).

Never pass a case that actually failed — report the real status from artifacts, even if the delta from expected is inconvenient.

### Step 5: Triage failures

For every `fail`:

- **Test bug** — flaky selector, wrong assertion, timing issue, outdated fixture. Evidence: selector changed but DOM intact; assertion mismatched but feature works by hand; race condition visible in trace.
- **Product bug** — real defect requiring code fix. Evidence: feature broken in the browser; API returns wrong shape; state mutation incorrect.
- **Infra issue** — CI environment, network flake, missing service, fixture drift. Evidence: container failed to start; external API timeout; test DB not migrated.

Each triage entry must cite evidence (stack trace excerpt, screenshot path, log line, HAR reference). No verdicts without evidence.

Assign a suggested fix owner:
- **Product bug** → dev team (backend or frontend per boundary).
- **Test bug** → QA.
- **Infra issue** → SRE / platform / CI owner.

Include reproduction steps (3–5 lines) so whoever picks it up can repro without the review author.

### Step 6: Coverage report (full mode only)

Compare actual coverage (from lcov / cobertura / pytest-cov JSON) against the numeric targets in `QA_STRATEGY.md`:

- Per module / package: lines %, branches %, functions %.
- Flag any area under target as a gap with a file-level breakdown.
- Call out modules with coverage but no assertions of value (vanity coverage — touched but not meaningfully verified).

### Step 7: Visual regression + a11y matrix (full only)

- **Visual regression:** summarize snapshot diffs (how many changed, how many approved vs new). List per-route diff paths. Flag any diff the reviewer cannot classify as intended.
- **A11y matrix:** per component × WCAG AA criterion. Columns: component, contrast, keyboard, SR labels, focus order, ARIA correctness. Cite axe violation IDs where present.

### Step 8: Perf budget (full only)

- Per route: LCP, INP, CLS measured vs budget from `QA_STRATEGY.md`.
- Per API endpoint: p50, p95, p99 latency vs budget.
- Any breach → flagged with magnitude (e.g., "LCP 3.4s vs budget 2.5s — 36% over").

### Step 9: Regression gap analysis

What should have been tested but isn't covered by `TEST_CASES.md`:

- FR-XXX in PRD with no matching case ID in the matrix.
- User journeys with coverage only at the happy path (no negative case).
- Components introduced in this build with no a11y / interaction test.
- Integrations (external API, third-party SDK) without contract test.
- Recent bug fixes without a regression test attached.

Surface each gap as a candidate entry for `qa-tasks` to add. Do not add tasks here — flag them for the next qa-tasks run.

### Step 10: Follow-up tasks

List new tasks to add to `QA_TASKS.md`:

- **Product bug fixes** — hand off to dev team (backend/frontend slice as appropriate).
- **Test bug fixes** — stay in QA.
- **Coverage gap plugs** — stay in QA.
- **Infra fixes** — hand off to SRE / platform.

Each follow-up carries: title, target owner, priority (blocker / high / normal), linked failure ID or gap reference.

### Step 11: Write QA_REVIEW.md

Save to `.qa/<feature>/QA_REVIEW.md`. The template uses a 4-backtick outer fence so nested code blocks (stack traces, Gherkin excerpts) render cleanly:

````markdown
# QA Review: [Feature Name]

**Feature:** [name]
**testingRigor:** [mvp | full]
**Review date:** [YYYY-MM-DD]
**Build SHA:** [short sha]
**Run URL:** [CI run link or local run path]
**Scope:** test execution results only — visual critique belongs to design-review / frontend-review

## Summary

**Status:** [green | yellow | red]

[1-sentence rationale: "All smoke E2E green, 2 unit failures triaged as test bugs — safe to release pending fixes." or "3 product bugs on critical path — do not release."]

## Pass/fail matrix

| Case ID | Title | Status | Duration | Last run |
|---------|-------|--------|----------|----------|
| TC-001 | User can sign in | pass | 2.1s | 2026-04-18 14:22 |
| TC-002 | Sign in with bad password | fail | 1.8s | 2026-04-18 14:22 |
| TC-003 | Password reset email | skipped | — | 2026-04-18 14:22 |
| TC-004 | Session persists across reload | missing | — | — |

**Totals:** [N pass] / [M fail] / [K skipped] / [J missing]

## Failures triage

### TC-002 — Sign in with bad password

- **Classification:** Test bug
- **Evidence:**
  ```
  Error: expect(received).toBe(expected)
  Expected: "Invalid credentials"
  Received: "Invalid email or password"
  at tests/e2e/signin.spec.ts:42:18
  ```
- **Reproduction:**
  1. Run `pnpm test:e2e signin.spec.ts`
  2. Observe assertion mismatch on error copy
  3. Manual sign-in with bad password shows correct error in UI
- **Suggested fix owner:** QA (update assertion to match product copy)
- **Follow-up:** QA-FOLLOWUP-001

### [next failure block...]

## Coverage report

_(full mode only — omitted in mvp)_

| Module | Lines | Branches | Functions | Target | Gap |
|--------|-------|----------|-----------|--------|-----|
| src/auth | 82% | 71% | 88% | 80/70/80 | — |
| src/billing | 64% | 48% | 70% | 80/70/80 | lines -16, branches -22 |

## Visual regression summary

_(full only)_

- 14 snapshots diffed, 2 changed, 2 approved as intended, 0 unexplained.
- Diff paths: `.playwright/snapshots/__diff_output__/...`.

## A11y matrix

_(full only)_

| Component | Contrast | Keyboard | SR labels | Focus order | ARIA | Notes |
|-----------|----------|----------|-----------|-------------|------|-------|
| Button | pass | pass | pass | pass | pass | |
| Modal | pass | fail | pass | fail | pass | focus trap missing (axe rule `focus-trap`) |

## Perf budget compliance

_(full only)_

| Route | LCP | INP | CLS | Budget | Status |
|-------|-----|-----|-----|--------|--------|
| / | 1.8s | 140ms | 0.02 | 2.5s/200/0.1 | pass |
| /dashboard | 3.4s | 180ms | 0.04 | 2.5s/200/0.1 | LCP over by 36% |

| Endpoint | p50 | p95 | Budget p95 | Status |
|----------|-----|-----|------------|--------|
| GET /v1/projects | 80ms | 220ms | 300ms | pass |
| POST /v1/invoices | 210ms | 640ms | 400ms | p95 over by 60% |

## Regression gaps

- FR-014 "Admin can export report" — no matching case in TEST_CASES.md.
- Happy path only for password reset — no negative case (bad token, expired token).
- New `InvoiceList` component — no a11y test.
- Stripe webhook integration — no contract test.

## Follow-up tasks

| ID | Title | Owner | Priority | Source |
|----|-------|-------|----------|--------|
| QA-FOLLOWUP-001 | Update TC-002 assertion to match product copy | QA | normal | Failures triage |
| QA-FOLLOWUP-002 | Add focus-trap to Modal component | frontend | high | A11y matrix |
| QA-FOLLOWUP-003 | Investigate /dashboard LCP regression | frontend | high | Perf budget |
| QA-FOLLOWUP-004 | Add E2E case for FR-014 export | QA | normal | Regression gaps |
````

### Step 12: Hand off

Summarize to the user:

> "Review complete. Status: [green|yellow|red]. [N product bugs / M test bugs / K infra issues]. Report saved at `.qa/<feature>/QA_REVIEW.md`. If status is **green** → release-ready per PRD acceptance. If **yellow** → decide with PM whether to ship. If **red** → fix-and-retest before release."

## Rules

- Never pass a test that actually failed — always report the real status from artifacts.
- Never invent test cases — the review is a report, not a creator. Gaps feed back to `qa-tasks`, not to QA_REVIEW.md case creation.
- Triage categorization must cite evidence (stack trace, screenshot, logs, HAR). No evidence → no verdict.
- Read `PROJECT_PROFILE.md` at Step 2 and branch — `mvp` skips Steps 6–8 entirely.
- Markdown only. Never modify test code or source. Recommend fixes; do not apply them.
- `testingRigor` governs the output shape — same skill, different depth.
- Defer visual fidelity to `design-review` and runtime engineering quality to `frontend-review`. This skill reports on test execution results, not aesthetics or runtime audits.
