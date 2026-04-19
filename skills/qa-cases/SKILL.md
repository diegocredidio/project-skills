---
name: qa-cases
description: Reads `.pm/<feature>/PRD.md` (FR-XXX + journeys), `.qa/<feature>/QA_STRATEGY.md`, and design component specs, then produces `.qa/<feature>/TEST_CASES.md` — structured Gherkin test cases (Given/When/Then) grouped by journey, component, and API endpoint. Coverage scope depends on `testingRigor` from PROJECT_PROFILE.md (`mvp` covers must-have FR only; `full` covers every FR plus edge and negative cases). Use when qa-flow invokes it or when user says "write the test cases", "gherkin the requirements". Not for implementing tests (that's qa-tasks) or for exploratory testing.
---

# QA Cases — Gherkin Test Case Generation

Transforms PRD requirements and user journeys into structured Given/When/Then scenarios.

## How to run

### Step 1: Load inputs

Read:
- `.pm/<feature>/PROJECT_PROFILE.md` — for `testingRigor` (`mvp` | `full`)
- `.pm/<feature>/PRD.md` — for FR-XXX items and user journeys
- `.qa/<feature>/QA_STRATEGY.md` — for pyramid decisions and tool choices (so cases land on the right layer)
- `.design/<feature>/COMPONENT_SPECS.md` (optional) — for a11y requirements per interactive component
- `.backend/<feature>/BACKEND_API.md` (optional) — for endpoint contracts to drive API-level cases

If PRD.md or QA_STRATEGY.md is missing, abort and direct the user to run the missing predecessor. If PROJECT_PROFILE.md is missing, fall back to the standard migration prompt (ask `designMode` and `testingRigor`).

### Step 2: Scope cases to testingRigor

Branch on `testingRigor`:

- **`mvp`** — cover only FR-XXX items marked "must" in the PRD. Skip edge cases. Skip negative paths unless the negative path IS a must (e.g., "login must reject invalid credentials" is a must, not an edge).
- **`full`** — cover every FR-XXX item regardless of priority, plus edge cases (boundary values, empty states, concurrent updates, network failure) plus explicit negative cases (invalid input, unauthorized access, rate limits).

Record the chosen scope in the TEST_CASES.md header so downstream readers know what was intentionally excluded.

### Step 3: Group cases by type

Every test case belongs to one of three buckets:

- **Journey-level E2E** — one scenario per critical user journey named in the PRD. Drives through the UI end-to-end. Owns the "happy path demonstrated in real browser" coverage.
- **Component-level** — one scenario per interactive component with a11y or state requirements (forms, modals, tables with sort/filter, keyboard-nav widgets). Runs in the component test runner (RTL, Vue Test Utils, etc.).
- **API-level** — one scenario per endpoint × state combination (success, validation error, auth error, not-found, rate-limit). Runs as API integration tests hitting the real server.

A single FR-XXX usually produces cases in 2–3 buckets; that's expected and desired.

### Step 4: Write Gherkin scenarios

Use canonical Gherkin shape:

```gherkin
Feature: [Feature name from PRD]

  Scenario: [Short descriptive name — one behavior]
    Given [starting context / preconditions]
    And [additional context]
    When [the action under test]
    Then [expected outcome]
    And [additional assertion]

  Scenario Outline: [Parameterized name]
    Given a user with role <role>
    When they attempt <action>
    Then the result is <outcome>

    Examples:
      | role    | action         | outcome     |
      | admin   | delete project | success     |
      | member  | delete project | forbidden   |
      | visitor | delete project | redirect    |
```

Concrete example (login journey, FR-001 "User can log in with email + password"):

```gherkin
Feature: Login

  Scenario: Successful login with valid credentials
    Given a registered user with email "alice@example.com"
    And the user is on the login page
    When they enter "alice@example.com" and the correct password
    And they submit the form
    Then they are redirected to the dashboard
    And the header shows "alice@example.com"

  Scenario: Rejected login with invalid password
    Given a registered user with email "alice@example.com"
    And the user is on the login page
    When they enter "alice@example.com" and an incorrect password
    And they submit the form
    Then they remain on the login page
    And an error "Invalid email or password" is shown
    And no session cookie is set
```

Rules for each scenario:
- `Scenario:` name describes one behavior, not a feature
- `Given` sets state, never performs the action under test
- `When` is a single action (use `And When` only for compound actions like "login then navigate")
- `Then` assertions are observable (visible text, URL, response shape) — not implementation details (no "a function was called")
- Use `Scenario Outline` + `Examples:` when the same flow repeats with different data

### Step 5: Build the coverage matrix

Produce a table mapping every FR-XXX in the PRD to its test case IDs:

| FR | Priority | Journey cases | Component cases | API cases |
|----|----------|---------------|-----------------|-----------|
| FR-001 | must | J-001, J-002 | C-001 | A-001, A-002 |
| FR-002 | must | J-003 | — | A-003 |
| FR-003 | should | — | — | — (uncovered — mvp scope) |

Any FR-XXX without at least one case ID must be flagged explicitly:
- In `mvp` mode: OK only if the FR is priority "should" or "could"; flag anything "must" that's uncovered as a scope bug.
- In `full` mode: every FR-XXX must have at least one case. Uncovered "must" or "should" items are bugs in this step.

### Step 6: Note flakiness / fragile selectors

Before writing cases, scan journeys for anti-patterns and call them out. Known fragile patterns:

- Selecting by generated CSS class names (e.g., `.css-1a2b3c4`) — breaks on every bundler change
- XPath or deep DOM paths (`div > div > div:nth-child(3)`) — breaks on layout changes
- Selecting by visible text in i18n projects — breaks on translation edits
- Implicit waits (`sleep(1000)`) instead of explicit waits for UI state
- Test ordering dependencies — Scenario B assumes Scenario A ran first

Prefer:
- `data-testid` attributes for E2E and component tests
- Role-based queries (`getByRole("button", { name: "Submit" })`) for a11y-aligned selectors
- Explicit wait conditions (`expect(locator).toBeVisible()`)
- Fresh fixtures per scenario — no shared mutable state

Document the selector strategy in the "Flakiness notes" section of TEST_CASES.md so downstream test authors don't reinvent it.

### Step 7: Write TEST_CASES.md

Save to `.qa/<feature>/TEST_CASES.md`:

`````markdown
# Test Cases: [Feature Name]

**testingRigor:** [mvp | full]
**Source:** PRD.md, QA_STRATEGY.md, COMPONENT_SPECS.md
**Date:** [YYYY-MM-DD]

## Coverage matrix

| FR | Priority | Journey | Component | API |
|----|----------|---------|-----------|-----|
| FR-001 | must | J-001, J-002 | C-001 | A-001 |
| FR-002 | must | J-003 | — | A-002 |
| FR-003 | should | — | — | — (mvp scope) |

**Uncovered FR-XXX flagged:** [none | list]

## Journey-level E2E cases

### J-001: Successful login
Covers: FR-001

```gherkin
Feature: Login

  Scenario: Successful login with valid credentials
    Given a registered user with email "alice@example.com"
    And the user is on the login page
    When they enter valid credentials and submit
    Then they are redirected to the dashboard
```

### J-002: Rejected login
Covers: FR-001

```gherkin
Feature: Login

  Scenario: Rejected login with invalid password
    Given a registered user with email "alice@example.com"
    When they enter an incorrect password and submit
    Then an error "Invalid email or password" is shown
```

## Component-level cases

### C-001: LoginForm — keyboard navigation
Covers: FR-001 (a11y requirement from COMPONENT_SPECS)

```gherkin
Feature: LoginForm accessibility

  Scenario: Tab order follows visual order
    Given the LoginForm is mounted
    When the user presses Tab repeatedly
    Then focus moves: email -> password -> submit -> forgot-password link
```

## API-level cases

### A-001: POST /auth/login — success
Covers: FR-001

```gherkin
Feature: Auth API

  Scenario: POST /auth/login returns 200 with valid credentials
    Given a user exists with email "alice@example.com"
    When POST /auth/login with valid body
    Then the response status is 200
    And the body includes a session token
    And the Set-Cookie header contains "session="
```

### A-002: POST /auth/login — validation error

```gherkin
Feature: Auth API

  Scenario Outline: POST /auth/login rejects invalid bodies
    When POST /auth/login with <body>
    Then the response status is <status>
    And the body includes error code <code>

    Examples:
      | body                     | status | code                |
      | missing email            | 422    | VALIDATION_ERROR    |
      | missing password         | 422    | VALIDATION_ERROR    |
      | malformed JSON           | 400    | BAD_REQUEST         |
```

## Negative cases

> **full mode only.** If this section appears in an `mvp` output, it's a misconfiguration — flag and ask.

### N-001: Rate-limit on login
Covers: FR-001 (implicit — NFR from ARCHITECTURE)

```gherkin
Feature: Auth API

  Scenario: POST /auth/login rate-limits after 5 failed attempts
    Given 5 failed login attempts from IP "1.2.3.4" in the last minute
    When POST /auth/login from "1.2.3.4" with any credentials
    Then the response status is 429
    And the Retry-After header is set
```

## Flakiness notes

- **Selectors:** all E2E cases use `data-testid` attributes. Component tests use `getByRole` where the accessible name is stable.
- **Waits:** no `sleep()` calls. Use `expect(locator).toBeVisible()` or equivalent explicit waits.
- **Fixtures:** each scenario seeds its own fixtures via the `beforeEach` hook. No shared mutable state across scenarios.
- **i18n:** visible-text assertions are wrapped in a `t("key")` helper so translation edits don't break tests.
- **Known fragile areas:** [list any flagged earlier — e.g., "the dashboard summary widget re-renders asynchronously; scenarios must wait for `data-testid=summary-loaded`"]
`````

### Step 8: Hand off

Tell the user: "Cases written. Next: `qa-tasks` decomposes these into vertical slices."

## Rules

- Always read `.pm/<feature>/PROJECT_PROFILE.md` first and branch coverage on `testingRigor`. No rigor = no cases.
- Never skip a "must" FR-XXX regardless of rigor. `mvp` trims "should" and "could"; it does not trim "must".
- Markdown-only output. This skill does not write test code into the source tree — that's `qa-tasks` and external implementation.
- Never invent FR-XXX that aren't in the PRD. If a case feels necessary but no FR maps to it, flag it in the coverage matrix as an uncovered concern and ask PM to add the FR.
- Flag fragile selectors proactively in the Flakiness notes section. Silent adoption of brittle patterns is the main cause of flaky test suites.
