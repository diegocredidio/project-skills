---
name: bdd-flow
description: Conducts a Three Amigos BDD session for a named feature, producing `.bdd/<feature>/BACKEND_FEATURES.md` and `.bdd/<feature>/FRONTEND_FEATURES.md` — Gherkin scenarios separated by discipline (API-level vs UI-level). Reads PRD.md and ARCHITECTURE.md as inputs; reads parent BDD artifacts if PARENT.md is present. Invoked from pm-handoff before discipline flows, or directly by the user after pm-architecture. Use when user says "BDD session", "write feature files", "Three Amigos", "gherkin the feature before we code". Not for writing step definitions (that's the dev team's job) or for ad-hoc test planning (use qa-cases).
---

# BDD Flow — Three Amigos Scenario Session

Orchestrates the BDD scenario session for a feature, then hands back.

## How to run

### Step 1: Verify inputs

Ask the user: "Which feature? (name of the `.pm/<feature>/` folder)"

Check that `.pm/<feature>/PRD.md` exists. If missing, abort with:

> "Run `pm-prd` first — I need the feature requirements."

Check that `.pm/<feature>/ARCHITECTURE.md` exists. If missing, abort with:

> "Run `pm-architecture` first — I need the system boundaries to scope scenarios."

Check for `.pm/<feature>/PARENT.md` — note presence (lineage mode) or absence (standalone). This flag is passed to `bdd-scenarios`.

### Step 2: Prepare output folder

Check if `.bdd/<feature>/` exists. If not, create it and tell the user:

> "Creating `.bdd/<feature>/`."

### Step 3: Invoke `bdd-scenarios`

Tell the user:

> "Starting Three Amigos session. I'll work through each FR with you."

Invoke `bdd-scenarios`.

### Step 4: Hand back

After `bdd-scenarios` completes, tell the user:

> "Feature files written to `.bdd/<feature>/`. You can now run `backend-flow` and `frontend-flow` — both will read these as behavioral contracts. `qa-cases` will also switch to augment mode and skip rewriting the scenarios."

## Rules

- Never write scenarios directly — that belongs to `bdd-scenarios`.
- Never proceed without both PRD.md and ARCHITECTURE.md present.
- Always create `.bdd/<feature>/` before invoking `bdd-scenarios`.
- If the feature is an evolution (PARENT.md present), tell `bdd-scenarios` — it will scope the session to new/modified FRs only.
