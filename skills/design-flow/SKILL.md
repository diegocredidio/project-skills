---
name: design-flow
description: Runs the full design workflow as a guided sequence — from brief intake through tokens, information architecture, components, tasks, and (post-build) design review. Orchestrates the design-* skills in order with confirmations between steps. Supports two entry points: Path A (a DESIGN_BRIEF.md from pm-handoff already exists — skip the grill) or Path B (starting from scratch — run design-grill first). Use when user says "design flow", "start the design", "run the design process", "design this feature", or when pm-handoff invokes this skill. Not for general design critique, free-form UI ideation, or one-off visual questions — those belong in a direct tool conversation, not this orchestrated flow.
---

# Design Flow — Full Design Workflow Orchestrator

Coordinates the in-package design skills as a guided sequence that ends with a handoff to `frontend-flow` (and optionally `design-review` post-build).

## Sequences

**Path A — `DESIGN_BRIEF.md` already exists (e.g., from pm-handoff):**
1. `design-brief-intake` — enrich brief with codebase audit
2. `design-ia` — information architecture
3. `design-tokens` — design token system
4. `design-components` — component specs
5. `design-tasks` — vertical slices for the frontend team
6. (external build step by frontend-flow)
7. `design-review` — post-build critique with screenshots

**Path B — Starting from scratch:**
1. `design-grill` — interrogate aesthetic direction and constraints
2. `design-brief-intake` — produce the canonical brief
3. `design-ia` — information architecture
4. `design-tokens` — design token system
5. `design-components` — component specs
6. `design-tasks` — vertical slices
7. (external build step)
8. `design-review`

## How to run

### Step 1: Setup

Check that `.design/` exists at the repo root. If not, create it.

Ask: "Which feature are we designing?" Use a kebab-case slug. If invoked by pm-handoff, the slug is already passed.

If `.design/<feature>/` exists with prior artifacts, show what's present and ask: "Continue where we left off, or start fresh?"

### Step 2: Pick the entry path

Check for `.design/<feature>/DESIGN_BRIEF.md`:

- **Exists (produced by pm-handoff)** → Path A. Announce: "Found DESIGN_BRIEF.md from pm-handoff — skipping design-grill, going straight to design-brief-intake to enrich with codebase audit."
- **Does not exist** → Path B. Announce: "Starting from scratch — running design-grill first to establish aesthetic direction."

### Step 3: Run each sub-skill in sequence

For each sub-skill in the chosen path:

1. Announce: "**Step N/[total]: [skill name]** — [one-line description]"
2. Invoke the skill
3. Save the output to `.design/<feature>/`
4. Summarize what was produced (artifact name + 2-3 highlights)
5. Ask: "Ready to continue? You can also skip, go back, stop here, or redo this step."

### Step 4: Handle navigation

- `next` / `continue` → proceed to next step
- `skip` → skip current step (flag artifact as missing in final summary)
- `back` / `go back to [step name]` → re-run the prior step
- `stop` → end and summarize what was produced
- `redo` → re-run current step with user feedback

### Step 5: Pre-build handoff

After `design-tasks` completes, pause with:

> "The design plan is complete. Artifacts are in `.design/<feature>/`. Now the frontend team can run `frontend-flow` — it will read TOKENS.md, IA.md, COMPONENT_SPECS.md, and DESIGN_TASKS.md alongside FRONTEND_BRIEF.md. Come back to `design-review` once the build is running."

### Step 6: Post-build review (on demand)

When the user returns with "run design-review" or similar:
- Verify the implementation is running (a local dev URL or route list)
- Invoke `design-review`
- Summarize the prioritized fix list

### Step 7: Final summary

```
## Design Flow Complete: [Feature Name]

### Planning artifacts (.design/<feature>/):
- ✅ DESIGN_GRILL.md (Path B only)
- ✅ DESIGN_BRIEF.md (enriched)
- ✅ CODEBASE_AUDIT.md
- ✅ IA.md
- ✅ TOKENS.md
- ✅ COMPONENT_SPECS.md
- ✅ DESIGN_TASKS.md
- ✅ DESIGN_REVIEW.md (post-build)

### What to do next:
- Run `frontend-flow` to implement using these artifacts
- Come back to `design-review` once a preview URL is running
```

## Rules

- Save outputs incrementally — stopping at step 3 still leaves steps 1–3 on disk.
- Every sub-skill reads inputs from `.design/<feature>/` files, not from conversation history — safe to resume across sessions.
- Respect the user's existing codebase — `design-brief-intake` runs a codebase audit and later skills honor the `CODEBASE_AUDIT.md` findings (existing tokens, existing components).
- On Path A, never re-ask decisions already resolved in the pm-handoff brief.
- `design-review` is terminal and may loop back to earlier skills only on user request (e.g., "redo tokens based on the review").
