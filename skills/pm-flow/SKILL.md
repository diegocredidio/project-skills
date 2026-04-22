---
name: pm-flow
description: Run the full PM workflow as a guided sequence for a concrete, named project or feature — from requirements grilling to task generation, plan review, and handoff to design/backend/frontend specialist flows. Orchestrates all PM skills in order with confirmations between steps. Supports two entry points: starting from scratch (full grill) or starting from an existing PRD/document (intake + gap-fill grill). Use when user says "pm flow", "plan a project", "start the pm process", "help me plan this feature", or shares a project idea that needs structured planning. Not for open-ended ideation (use superpowers:brainstorming) or generic multi-step plan authoring (use superpowers:writing-plans) — this skill only runs when the subject has a nameable scope and the user wants PM artifacts produced in `.pm/<feature>/`.
---

# PM Flow — Full Workflow Orchestrator

Run the complete PM planning process as a guided sequence. Supports two entry paths and ends with handoff to specialist skill-sets (design, backend, frontend).

## Sequences

**Path A — Starting from scratch:**
1. `pm-grill` — Full requirements interrogation
2. `pm-prd` — PRD generation
3. `pm-architecture` — Technical architecture
4. `pm-workstreams` — Workstream breakdown (back/front/design)
5. `pm-tasks` — Detailed task generation
6. `pm-review` — Plan review and gap analysis
7. `pm-handoff` — Generate briefings + invoke specialist skill-sets

**Path B — Starting from existing material:**
1. `pm-intake` — Ingest document, produce gap analysis
2. `pm-grill` (gap-fill mode) — Interrogate only the identified gaps
3. `pm-prd` — PRD generation
4. `pm-architecture` — Technical architecture
5. `pm-workstreams` — Workstream breakdown
6. `pm-tasks` — Detailed task generation
7. `pm-review` — Plan review and gap analysis
8. `pm-handoff` — Generate briefings + invoke specialist skill-sets

## How to run

### Step 1: Setup

Check if `.pm/` directory exists. If not, create it.

Scan `.pm/` for existing slugs. For each existing slug, compute a readiness indicator by checking:
- `PRD.md` — present in `.pm/<slug>/` → emit `PRD ✓`
- `ARCHITECTURE.md` — present in `.pm/<slug>/` → emit `ARCHITECTURE ✓`
- Any of `.design/<slug>/DESIGN_BRIEF.md`, `.backend/<slug>/BACKEND_BRIEF.md`, `.frontend/<slug>/FRONTEND_BRIEF.md`, `.qa/<slug>/QA_BRIEF.md` → emit `handed off`

If at least one slug exists, list each with its readiness indicator and ask (showing one `evolution of <slug>` option per enumerated slug — not a literal `<slug>` placeholder):

> "Existing features under `.pm/`:
> - auth (PRD ✓, ARCHITECTURE ✓, handed off)
> - dashboard (PRD ✓, no handoff yet)
>
> Is this a new feature or an evolution of one of those?
> [new / evolution of auth / evolution of dashboard / cancel]"

**If the user picks `evolution of <slug>`**, invoke `pm-extend` with `parent=<slug>`. Ask the user for the child slug name and the improvement summary, then let `pm-extend` run its full flow (Steps 1–8). pm-extend ends by invoking `pm-flow` on the child slug, which re-enters this Step 1 and matches the "subfolder exists with PARENT.md" branch below. Do NOT proceed to Step 2 in the current invocation.

**If the user picks `new`** (or `.pm/` is empty), ask: "What feature or project do you want to plan?" Create a subfolder name from the answer (kebab-case). Then check the subfolder state:

- **Subfolder exists AND contains `PARENT.md`** → skip the existing-material prompt and continue to Step 2 — the `PARENT.md` check there will auto-select Path B.
- **Subfolder exists WITHOUT `PARENT.md`** → show what's there and ask: "Continue from where you left off, or start fresh?"
- **Subfolder does not exist** → create it and continue to Step 2.

### Step 2: Choose entry point

If `.pm/<feature>/PARENT.md` exists, skip the prompt and go directly to Path B (pm-intake already ran — the seeded INTAKE.md from pm-extend is the "existing material"). Announce:

> "Lineage detected — `<feature>` is an evolution of `<parent>`. Skipping the existing-material prompt and running Path B with the seeded INTAKE.md."

Otherwise:

> "Do you have existing material — a PRD draft, brief, requirements list, or proposal? Or start from scratch?"

**Existing material → Path B:** Run `pm-intake`, then `pm-grill` in gap-fill mode.

**From scratch → Path A:** Run `pm-grill` in full mode.

### Step 3: Run each skill in sequence

For each skill:
1. Announce: "**Step N/[total]: [Skill Name]** — [one-line description]"
2. Execute the skill
3. Save output to `.pm/<feature-name>/`
4. Show summary of what was produced
5. Ask: "Ready to continue? You can also skip, go back, or stop here."

### Step 4: Handle navigation

- "next" / "continue" → proceed
- "skip" → skip to next step
- "back" / "go back to [step]" → re-run previous step
- "stop" → end, summarize produced
- "redo" → re-run current step

### Step 5: Handoff confirmation

Before running `pm-handoff`, confirm:
> "PM planning is complete. Ready to hand off to the specialist skill-sets? I'll generate briefings for design, backend, and frontend — you can choose which to start."

### Step 6: Final summary

```
## PM Flow Complete: [Feature Name]

### Planning artifacts (.pm/):
- ✅ INTAKE.md (Path B only)
- ✅ GRILL_SUMMARY.md
- ✅ PRD.md
- ✅ ARCHITECTURE.md
- ✅ WORKSTREAMS.md
- ✅ TASKS.md
- ✅ REVIEW.md

### Execution briefings:
- ✅ .design/<feature>/DESIGN_BRIEF.md  → design-flow
- ✅ .backend/<feature>/BACKEND_BRIEF.md → backend-flow
- ✅ .frontend/<feature>/FRONTEND_BRIEF.md → frontend-flow

### What to do next:
- Start design-flow first — frontend needs tokens/specs before components
- Run backend-flow in parallel with design-flow
- Start frontend-flow once design specs and backend contracts are ready
```

## Rules

- Save outputs incrementally — stopping at step 3 still gives them steps 1–3.
- Each step reads from previous output files, not conversation history.
- Explore existing codebase before generating anything.
- On Path B, never re-ask things already resolved in the intake.
- The handoff step is always last — it needs all PM artifacts to be complete.
- On evolution (user picked "evolution of <slug>" or `.pm/<feature>/PARENT.md` exists), `pm-extend` owns Step 1's child-bootstrap work and `pm-flow` resumes from Step 2 in Path B mode. Do not re-ask for the parent slug mid-flow.
