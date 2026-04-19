---
name: qa-flow
description: Orchestrates the full QA definition pipeline for a named project or feature — intake → strategy → cases → tasks → review. Reads `.qa/<feature>/QA_BRIEF.md` (from pm-handoff) or asks the user to paste one, then chains the specialist qa-* skills with confirmations between steps. Use when user says "qa flow", "start the qa", "plan the testing", or when pm-handoff invokes this skill. Not for ad-hoc test debugging, generic QA consulting, or single-step work — each sub-step has its own skill (qa-brief-intake, qa-strategy, qa-cases, qa-tasks, qa-review).
---

# QA Flow — Orchestrator

Coordinates the in-package qa-* skills as a guided sequence.

## Sequence

1. `qa-brief-intake` — ingest QA_BRIEF.md + audit existing test infra + gap-fill grill
2. `qa-strategy` — test pyramid, tools by level, coverage targets, risk matrix, test data strategy
3. `qa-cases` — PRD FR-XXX + user journeys → Gherkin / Given-When-Then test cases
4. `qa-tasks` — vertical-sliced QA task list
5. (external implementation and build)
6. `qa-review` — post-build test execution report + failure triage + regression gaps

## How to run

### Step 1: Setup

Check that `.qa/<feature>/` exists. If invoked by pm-handoff, the slug is already passed. Otherwise ask: "Which feature/project?" Use a kebab-case slug.

If artifacts already exist, show what's present and ask: "Continue from where we left off, or start fresh?"

### Step 1.5: Read the project profile

Read `.pm/<feature>/PROJECT_PROFILE.md` to get `testingRigor` and `designMode`. If the file is missing:

> "Este projeto não tem PROJECT_PROFILE.md. Modo de design: **shadcn-theme** ou **custom-system**? Rigor de teste: **mvp** ou **full**? (Respostas gravam em `.pm/<feature-name>/PROJECT_PROFILE.md` — afetam design-tokens, design-components, qa-strategy, qa-review.)"

Write the file with the answers. Then announce the mode:

> "Rodando qa-flow em modo **<testingRigor>**. `qa-strategy` e `qa-review` vão se ajustar a esse rigor."

### Step 2: Locate the brief

Look for `.qa/<feature>/QA_BRIEF.md`. If absent, fall back to `.pm/<feature>/PRD.md` + `.pm/<feature>/ARCHITECTURE.md`. If neither exists, tell the user: "I need either a QA_BRIEF.md from pm-handoff or at least a PRD + architecture pair."

### Step 3: Run each sub-skill in sequence

For each step:

1. Announce: "**Step N/4: [skill name]** — [one-line description]"
2. Invoke the skill
3. Save output to `.qa/<feature>/`
4. Summarize what was produced
5. Ask: "Ready to continue? You can also skip, go back, stop, or redo."

### Step 4: Navigation

- `next` / `continue` → proceed
- `skip` → flag artifact as missing and continue
- `back` → re-run prior step
- `redo` → re-run current step
- `stop` → end, summarize what's produced

### Step 5: Pre-review gate

After `qa-tasks` completes, pause:

> "QA plan complete. Hand this to the QA team (or the devs, if solo) to implement per QA_TASKS.md. Come back to `qa-review` once the build is running and tests are executing."

### Step 6: Post-build review

When the user returns with "qa review" or similar, invoke `qa-review`.

### Step 7: Final summary

```
## QA Flow Complete: [Feature]

### Artifacts (.qa/<feature>/):
- ✅ QA_INTAKE.md
- ✅ QA_STRATEGY.md
- ✅ TEST_CASES.md
- ✅ QA_TASKS.md
- ✅ QA_REVIEW.md (post-build)

### What to do next:
- Implement automated tests per QA_TASKS.md
- Coordinate E2E test runs with backend and frontend workstreams
- Return for `qa-review` once the build is up and tests have executed
```

## Rules

- Save outputs incrementally — stopping at step 3 still leaves steps 1–3.
- Every sub-skill reads inputs from disk, not conversation history — safe to resume across sessions.
- Always read `.pm/<feature>/PROJECT_PROFILE.md` at Step 1.5. `testingRigor` changes what `qa-strategy` and `qa-review` produce.
- This skill does not execute tests. It plans them. Execution happens outside the flow.
- If the brief is thin, `qa-brief-intake` grills the gaps — do not bypass that by jumping to `qa-strategy`.
