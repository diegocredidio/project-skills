---
name: pm-grill
description: Scoped PM requirements interrogation for a named project or feature — produces `.pm/<feature>/GRILL_SUMMARY.md`. Operates in two modes: full mode (starting from scratch — walks every decision branch) or gap-fill mode (starting from an existing document — asks only the questions flagged by pm-intake). Use when user says "grill me", "stress test my plan", "challenge my requirements", or when the pm-flow orchestrator triggers this step. Not for general Socratic questioning, writing coaching, or free-form discovery — those belong to superpowers:brainstorming.
---

# PM Grill — Requirements Interrogation

Stress-test the project requirements until every ambiguity is resolved. Operates in two modes depending on whether material already exists.

---

## Mode detection

Before starting, check for `.pm/<feature-name>/INTAKE.md`:

- **If INTAKE.md exists** → run in **gap-fill mode** (Step A below)
- **If no INTAKE.md** → run in **full mode** (Step B below)

---

## Mode A: Gap-fill mode (existing document)

Use this when `pm-intake` has already run and produced an `INTAKE.md`.

### Step A1: Load the gap list

Read `.pm/<feature-name>/INTAKE.md` and extract the "Grill question list" section.

Tell the user:
> "I've reviewed your document. [N] things are already clear. I have [N critical + N important] questions to work through with you. Let's go."

Do NOT re-ask or re-verify the elements marked ✅ Clear in the intake. Respect the user's existing work.

### Step A2: Ask only the flagged questions

Work through the question list in priority order: Critical → Important → Minor.

For each question:
- Ask it specifically and concisely
- Wait for the answer
- Probe for specificity if the answer is vague — don't accept "it depends" without asking "depends on what?"
- If the user's answer introduces a new ambiguity, follow it up before moving on
- When resolved, confirm: "Got it — [summarize the decision]. Next question."

If the user's answer contradicts something in the original document, surface it: "You said X in the brief, but now you're saying Y. Which should we go with?"

### Step A3: Check for emergent gaps

After working through all flagged questions, do a quick scan:
- Did any answers reveal new questions that weren't in the original intake?
- Did any answers change the scope in a way that creates new open questions?

If yes, ask those too. If no, close the grill.

### Step A4: Produce the grill summary

Write `.pm/<feature-name>/GRILL_SUMMARY.md` with ALL resolved information — both what came from the original document (already clear) and what was resolved in the grill. Also write `.pm/<feature-name>/PROJECT_PROFILE.md` per the "Project Profile" section below.

---

## Mode B: Full mode (starting from scratch)

Use this when no intake document exists and the user is starting fresh.

### Step B1: Get the subject

If no plan or idea was provided, ask: "What project or feature do you want to build? Give me as much context as you have — even a sentence is fine to start."

### Step B2: Map the decision tree

Before asking anything, identify the main branches that need resolution. Present the map to the user:

> "Here's what I want to cover. I'll go branch by branch."

**Product branches:**
- Problem: What specific problem? For whom? How do they handle it today?
- Users: Who are the primary users? What's their context and technical level?
- Success metrics: What numbers need to move? How will we know this worked?
- Scope: What's in v1? What's explicitly out? MVP vs. full vision?
- User journeys: Walk the critical paths step by step
- Edge cases: What happens when things go wrong? Empty states, errors, limits?
- Alternatives: What exists today? Why isn't it enough?

**Technical branches:**
- Data: What data is needed? Source of truth? Ownership?
- Integrations: External systems, APIs, auth providers, payments?
- Scale: How many users? How much data? Growth trajectory?
- Constraints: Compliance? Accessibility? Performance? Budget?
- Existing codebase: What already exists that we need to respect or integrate with?
- Delivery profile: Design mode — `shadcn-theme` or `custom-system`? UI framework — `shadcn/ui`, other, or none?

**Organizational branches:**
- Team: Who's building? What skills are available?
- Timeline: When does this ship? Hard deadlines?
- Dependencies: What needs to happen first? What's blocked on this?
- Stakeholders: Who approves? Who has veto power?

Only map the branches that are genuinely relevant. Skip what doesn't apply.

### Step B3: Walk the tree systematically

Start with the most foundational branch — the one other decisions depend on (usually Problem or Users).

Within each branch:
- Ask one question at a time
- Wait for the answer
- Probe until the answer is specific and actionable: "What do you mean by 'scalable'?", "When you say 'users', who specifically?"
- Push back on vagueness without being aggressive
- When stuck, offer options: "Based on what you've told me, I'd suggest X because Y. Does that work?"
- Confirm when each branch is resolved: "Got it — [summarize]. Moving on."

### Step B4: Track and surface contradictions

Keep a running model of what's decided. If a later answer conflicts with an earlier one, call it out immediately: "Earlier you said X, but this implies Y. Which is it?"

### Step B5: Produce the grill summary

---

## Grill Summary (both modes)

Write `.pm/<feature-name>/GRILL_SUMMARY.md`:

```markdown
# Grill Summary: [Feature/Project Name]

**Mode:** [Full / Gap-fill]
**Date:** [Today]
**Source document:** [filename, if gap-fill mode]

---

## Problem
[2-3 sentences. What problem, for whom, how painful.]

## Target Users
[Specific. Roles, context, technical level.]

## Success Metrics
1. [Metric with target]
2. ...

## Scope — v1
### In scope
- [Item]
### Out of scope
- [Item]

## Key User Journeys
### Journey 1: [Name]
1. [Step]
2. [Step]
...

## Technical Constraints
- [Constraint]

## Organizational Constraints
- [Constraint]

## Open Questions
| # | Question | Recommended resolution | Owner |
|---|----------|----------------------|-------|
| 1 | [Question that couldn't be resolved] | [Suggestion] | [Who should decide] |

## Key Decisions Made
| Decision | Choice | Rationale |
|----------|--------|-----------|
| [Decision] | [What was decided] | [Why] |
```

---

## Project Profile (both modes)

When the Delivery profile branch is resolved, ALSO write `.pm/<feature-name>/PROJECT_PROFILE.md` — a minimal, machine-readable record that downstream skills (`pm-handoff`, `design-flow`, `design-tokens`, `design-components`, `frontend-stack`, `frontend-components`) read with a simple grep. Separate file from `GRILL_SUMMARY.md` so it survives re-runs of the grill.

Format:

```markdown
# Project Profile
designMode: shadcn-theme | custom-system
uiFramework: shadcn/ui | <other> | none
```

If the user didn't resolve the Delivery profile branch (gap-fill mode where the source doc pre-answered it), still write the file using the inferred values and cite the source in the GRILL_SUMMARY Key Decisions table.

## Rules

- Gap-fill mode: NEVER re-ask things marked ✅ Clear in the intake. It wastes the user's time and signals you didn't read their document.
- Full mode: the grill should take 10-15+ exchanges minimum. If it's done in 5, you weren't thorough enough.
- Never accept "it depends" without following up with "depends on what, specifically?"
- Never accept "we'll figure it out later" for foundational decisions. Name the cost: "If we don't decide this now, it means [specific consequence]."
- If something can be answered by exploring the codebase, explore it instead of asking.
- Offer a recommended answer when the user is stuck — give them something to react to.
- The grill summary is the source of truth for all downstream skills. Make it complete.
