---
name: backend-flow
description: Orchestrates the full backend definition pipeline for a named project or feature — intake → stack → data → api → tasks → review. Reads `.backend/<feature>/BACKEND_BRIEF.md` (from pm-handoff) or asks the user to paste one, then chains the specialist backend-* skills with confirmations between steps. Use when user says "backend flow", "start the backend", "plan the backend", or when pm-handoff invokes this skill. Not for ad-hoc stack debates, general backend consulting, or single-step work — each sub-step has its own skill (backend-stack, backend-data, backend-api, backend-tasks, backend-review).
---

# Backend Flow — Orchestrator

Coordinates the in-package backend skills as a guided sequence.

## Sequence

1. `backend-brief-intake` — ingest BACKEND_BRIEF.md + audit codebase + gap-fill grill
2. `backend-stack` — language, framework, DB, ORM, auth, structure, conventions
3. `backend-data` — data model, schema, migrations strategy
4. `backend-api` — endpoint contracts, OpenAPI/GraphQL/tRPC artifact
5. `backend-tasks` — vertical-sliced task list
6. (external implementation)
7. `backend-review` — static analysis vs declared specs

## How to run

### Step 1: Setup

Check that `.backend/<feature>/` exists. If invoked by pm-handoff, the slug is already passed. Otherwise ask: "Which feature/project?" Use a kebab-case slug.

If artifacts already exist, show what's present and ask: "Continue from where we left off, or start fresh?"

### Step 2: Locate the brief

Look for `.backend/<feature>/BACKEND_BRIEF.md`. If absent, fall back to `.pm/<feature>/ARCHITECTURE.md`. If neither, tell the user: "I need either a BACKEND_BRIEF.md from pm-handoff or at least a paste of the backend requirements."

### Step 3: Run each sub-skill in sequence

For each step:

1. Announce: "**Step N/6: [skill name]** — [one-line description]"
2. Invoke the skill
3. Save output to `.backend/<feature>/`
4. Summarize what was produced
5. Ask: "Ready to continue? You can also skip, go back, stop, or redo."

### Step 4: Navigation

- `next` / `continue` → proceed
- `skip` → flag artifact as missing and continue
- `back` → re-run prior step
- `redo` → re-run current step
- `stop` → end, summarize what's produced

### Step 5: Pre-review gate

After `backend-tasks` completes, pause:

> "Backend plan is complete. Hand this to a dev team to implement per BACKEND_TASKS.md. Come back to `backend-review` once code is landing for static analysis against the declared specs."

### Step 6: Post-build review

When the user asks for "backend review" or similar, invoke `backend-review`.

### Step 7: Final summary

```
## Backend Flow Complete: [Feature]

### Artifacts (.backend/<feature>/):
- ✅ BACKEND_INTAKE.md
- ✅ BACKEND_STACK.md
- ✅ BACKEND_DATA.md
- ✅ BACKEND_API.md
- ✅ BACKEND_TASKS.md
- ✅ BACKEND_REVIEW.md (post-build)

### What to do next:
- Implement tasks per BACKEND_TASKS.md
- Frontend can consume the contract from BACKEND_API.md (OpenAPI/GraphQL SDL)
- Return for `backend-review` once build is underway
```

## Rules

- Save outputs incrementally — stopping at step 3 still leaves steps 1–3.
- Every sub-skill reads inputs from disk, not conversation history — safe to resume across sessions.
- This skill does not generate code. Every sub-skill produces markdown + referenced contracts (OpenAPI, schemas). Implementation happens outside the flow.
- If the brief is thin or missing key sections (auth, data model), `backend-brief-intake` will grill — do not bypass that by jumping to `backend-stack`.
