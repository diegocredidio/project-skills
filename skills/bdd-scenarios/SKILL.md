---
name: bdd-scenarios
description: Conducts an interactive Three Amigos BDD session for a named feature. Reads PRD.md, ARCHITECTURE.md, and (if lineage) parent BDD feature files, then facilitates a per-FR question-and-answer session covering PM, Dev, and QA perspectives. Produces `.bdd/<feature>/BACKEND_FEATURES.md` (API-level Gherkin) and `.bdd/<feature>/FRONTEND_FEATURES.md` (UI-level Gherkin). Invoked by bdd-flow. Not for standalone use — use bdd-flow as the entry point.
---

# BDD Scenarios — Three Amigos Session Conductor

Drives the interactive Three Amigos conversation with the user, FR by FR, and writes the resulting Gherkin feature files.

## How to run

### Step 1: Load inputs

Read:
- `.pm/<feature>/PRD.md` — FR-XXX list with priorities and user journeys
- `.pm/<feature>/ARCHITECTURE.md` — system boundaries, API shape, layers

Lineage check: if `.pm/<feature>/PARENT.md` exists:
- Read the H1 to extract `<parent-slug>` (format: `# Parent: <parent-slug>`)
- Check for `.bdd/<parent-slug>/BACKEND_FEATURES.md` — load if present
- Check for `.bdd/<parent-slug>/FRONTEND_FEATURES.md` — load if present
- If parent BDD files exist: announce "Lineage detected. Parent `<parent-slug>` has existing BDD scenarios — I'll only run the session for new or modified FRs."
- If parent BDD files absent: announce "Lineage detected but parent `<parent-slug>` has no BDD artifacts — running full session for all FRs."

### Step 2: Scope the session

Build the FR list from PRD.md.

If lineage AND parent BDD files exist:
- Identify FRs from the PRD `## 0. Extends` table marked as `unchanged` → exclude from session (inherited)
- Identify FRs marked as `extends`, `modifies`, or `new` → include in session
- Show the user a summary table:
  ```
  Session scope:
  ✅ Inherited (no session needed): FR-001, FR-002
  🔄 To cover in session: FR-003 (modifies), FR-004 (new), FR-005 (new)
  ```

If no lineage (or no parent BDD files): include all FRs.

Group FRs by user journey where possible — FRs that share the same journey are batched into one session round to avoid repetition.

### Step 3: Announce session structure

Tell the user:

> "I'll work through [N] FRs (or FR groups). For each one I'll ask three rounds of questions:
> 1. PM perspective — business rules and boundaries
> 2. Dev perspective — contracts and constraints
> 3. QA perspective — negative paths and edge cases
>
> Then I'll draft backend and frontend scenarios for your review. Let's start."

### Step 4: Three Amigos session (repeat per FR or FR group)

For each FR (or group), do the following in order:

**4a. Present the FR:**

Show:
```
--- FR-XXX: [Title] ---
[Description from PRD]
Priority: [must / should / could]
```

**4b. PM perspective:**

Ask exactly:

> "PM lens — What is the exact business rule here? What must NOT happen (the boundary that, if crossed, means the feature is broken)?"

Wait for the user's answer. If the answer is vague (e.g. "it should work correctly", one sentence with no specifics), ask one follow-up:

> "Can you be more specific? For example: 'User can only do X if Y is true' or 'The system must reject Z under condition W'."

**4c. Dev perspective:**

Ask exactly:

> "Dev lens — What is the expected API contract (endpoint, method, key request/response fields)? Any technical constraint or invariant the implementation must respect?"

Wait for answer. One follow-up if vague.

**4d. QA perspective:**

Ask exactly:

> "QA lens — What is the critical negative path? What's the edge case that always gets forgotten for this kind of feature?"

Wait for answer. One follow-up if vague.

**4e. Draft scenarios:**

From the three answers, draft:
- 1–3 **backend scenarios** (API-level): request → response shape. No UI references.
- 1–3 **frontend scenarios** (UI-level): user action → visible outcome. No API endpoint references.

Use canonical Gherkin shape:

```gherkin
Scenario: [Short descriptive name — one behavior]
  Given [starting context]
  When [the action under test]
  Then [observable expected outcome]
```

Rules for scenarios:
- `Scenario:` name = one behavior, not a feature
- `Given` sets state, never performs the action under test
- `When` is a single action
- `Then` assertions are observable (visible text, URL, response shape) — not implementation details
- Backend: use HTTP verbs, status codes, response body shape. Never mention UI elements.
- Frontend: use user-visible actions (clicks, form submissions, navigation). Never mention API endpoints.

**4f. Present drafts and iterate:**

Show:
```
Backend scenarios (API-level):
[scenarios]

Frontend scenarios (UI-level):
[scenarios]

Look good? (yes / suggest changes)
```

If user suggests changes: revise and re-present. Repeat until user says yes.

**4g. Confirm and move on:**

Once approved, note the final scenarios and move to the next FR.

### Step 5: Handle inherited scenarios (lineage only)

For FRs classified as `unchanged` (excluded from the session):
- Do NOT generate new scenarios
- Record a reference comment for the output files:
  `<!-- FR-XXX: inherited from <parent-slug> — see .bdd/<parent-slug>/BACKEND_FEATURES.md -->`
  (same comment for FRONTEND_FEATURES.md but pointing to the frontend file)

### Step 6: Write BACKEND_FEATURES.md

Save to `.bdd/<feature>/BACKEND_FEATURES.md` using this template (write in actual Markdown, not as a code block — this is the file content):

```
# Backend Feature Scenarios: [Feature Name]

**Feature:** [feature slug]
**Source:** PRD.md, ARCHITECTURE.md
**Date:** [YYYY-MM-DD]
**Lineage:** [standalone | evolution of <parent-slug>]

---

## FR-XXX: [FR Title]

```gherkin
Feature: [Feature name]

  Scenario: [name]
    Given [context]
    When [action]
    Then [outcome]
```

<!-- FR-YYY: inherited from <parent-slug> — see .bdd/<parent-slug>/BACKEND_FEATURES.md -->

---

## Coverage index

| FR | Priority | Scenarios |
|----|----------|-----------|
| FR-XXX | must | [scenario names] |
| FR-YYY | must | inherited |
```

### Step 7: Write FRONTEND_FEATURES.md

Save to `.bdd/<feature>/FRONTEND_FEATURES.md` with the same header structure and coverage index, using UI-level scenarios.

### Step 8: Hand back to bdd-flow

Report: "Session complete. [N] FRs covered, [M] inherited from parent. Files written to `.bdd/<feature>/`."

---

## Rules

- Never skip the three-perspective question sequence for any FR in scope. All three lenses are required.
- Never guess on vague answers — always ask exactly one follow-up before drafting.
- Backend scenarios: never reference UI elements. Frontend scenarios: never reference API endpoints or implementation details.
- Inherited FRs get reference comments only — never copy-paste parent scenarios into child files.
- Coverage index at the end of each file is mandatory. Every FR in scope must appear in the index.
- Session proceeds FR by FR, waiting for user approval at each step before moving on.
- Gherkin in the output files is plain Gherkin inside markdown fenced blocks (```gherkin). Framework-agnostic — no Cucumber/Behave/SpecFlow boilerplate.

---

## Context

This is the most complex skill in the bdd-flow. It is an interactive, conversational skill — it drives a back-and-forth dialogue with the user, not a one-shot generation.

Read `skills/qa-cases/SKILL.md` for style reference on how the pack writes Gherkin scenarios. Read `skills/backend-brief-intake/SKILL.md` to understand the interactive grill-session pattern (similar conversation structure).

The skill is framework-agnostic: it writes Gherkin feature content, not step definition code.
