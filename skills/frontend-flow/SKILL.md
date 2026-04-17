---
name: frontend-flow
description: Orchestrates the full frontend definition pipeline for a named project or feature — intake → stack → routing → components → tasks → review. Reads `.frontend/<feature>/FRONTEND_BRIEF.md` (from pm-handoff) and design artifacts at `.design/<feature>/` (TOKENS.md, IA.md, COMPONENT_SPECS.md, DESIGN_TASKS.md), then chains the specialist frontend-* skills. Use when user says "frontend flow", "start the frontend", "plan the frontend", or when pm-handoff invokes this skill. Not for generic React questions, CSS troubleshooting, or single-file edits — each sub-step has its own skill.
---

# Frontend Flow — Orchestrator

Coordinates the in-package frontend skills as a guided sequence.

## Sequence

1. `frontend-brief-intake` — ingest FRONTEND_BRIEF.md + design artifacts + audit codebase + gap-fill grill
2. `frontend-stack` — framework, styling, state, forms, lib, testing, hosting
3. `frontend-routing` — translate design IA.md into concrete routes / layouts / boundaries
4. `frontend-components` — map design COMPONENT_SPECS.md to implementation plans
5. `frontend-tasks` — vertical slices, depends on design-tasks
6. (external implementation)
7. `frontend-review` — post-build: a11y / perf / responsive / bundle

## How to run

### Step 1: Setup

Confirm `.frontend/<feature>/` exists. If invoked by pm-handoff, slug already provided. Otherwise ask: "Which feature?" Kebab-case.

### Step 2: Locate brief + probe design artifacts

Look for `.frontend/<feature>/FRONTEND_BRIEF.md`. If absent, fall back to `.pm/<feature>/ARCHITECTURE.md`. If neither, ask for a paste.

Probe `.design/<feature>/`:
- `DESIGN_BRIEF.md` (enriched)
- `TOKENS.md`
- `IA.md`
- `COMPONENT_SPECS.md`
- `DESIGN_TASKS.md`

Warn on any missing — continue anyway, but flag downstream skills (`frontend-routing` needs `IA.md`, `frontend-components` needs `COMPONENT_SPECS.md` + `TOKENS.md`).

### Step 3: Run each sub-skill in sequence

For each:

1. Announce: "**Step N/6: [skill name]** — [one-liner]"
2. Invoke the skill
3. Save output to `.frontend/<feature>/`
4. Summarize
5. Ask navigation: next / skip / back / redo / stop

### Step 4: Pre-build handoff

After `frontend-tasks` completes:

> "Frontend plan complete. A dev team can implement per FRONTEND_TASKS.md using FRONTEND_STACK + FRONTEND_ROUTES + COMPONENT_PLAN. Return to `frontend-review` once a preview URL is running."

### Step 5: Post-build review

On user request, invoke `frontend-review`.

### Step 6: Final summary

```
## Frontend Flow Complete: [Feature]

### Artifacts (.frontend/<feature>/):
- ✅ FRONTEND_INTAKE.md
- ✅ FRONTEND_STACK.md
- ✅ FRONTEND_ROUTES.md
- ✅ COMPONENT_PLAN.md
- ✅ FRONTEND_TASKS.md
- ✅ FRONTEND_REVIEW.md (post-build)

### What to do next:
- Implement per FRONTEND_TASKS.md
- Coordinate with design-tasks dependencies (each frontend task references DESIGN_TASK IDs)
- Return for `frontend-review` once preview is up
```

## Rules

- If design artifacts are missing, flag it early — `frontend-components` and `frontend-routing` can degrade, but the user must know.
- Every sub-skill reads from disk, not conversation history — safe to resume.
- This skill does not generate code. Every sub-skill produces markdown / contract plans. Implementation happens outside.
- Do not duplicate design work — read design artifacts; do not reinvent tokens or component specs in frontend files.
