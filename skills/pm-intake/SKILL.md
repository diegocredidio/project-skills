---
name: pm-intake
description: Ingest an existing PRD, briefing document, requirements list, or project description and produce a structured gap analysis. Classifies every element as clear, ambiguous, or missing, then prepares a focused question list for the grill step. Use when user says "I have a PRD", "I have a brief", "analyze this document", "here are my requirements", "review this spec", or when the pm-flow orchestrator detects existing material. Also trigger when someone uploads or pastes a project document and wants to know what's missing before moving forward.
---

# PM Intake — Document Ingestion and Gap Analysis

Read the user's existing material, extract everything that's already clear, identify what's ambiguous or incomplete, and prepare a targeted question list for the grill. The grill should only ask about what this step flags — nothing else.

## How to run

### Step 1: Receive the document

Accept the input in any of these forms:
- A pasted document in the conversation
- A file upload (read it from the uploaded path)
- A file path in the project (e.g., `docs/brief.md`, `requirements.pdf`)
- A URL the user provides

If no document has been provided yet, ask: "Please share the document — you can paste it here, upload a file, or give me a file path in the project."

### Step 2: Parse the document

Read the entire document and extract structured information across these categories:

**Product categories:**
- Problem statement — what problem is being solved
- Target users — who the users are
- Goals and success metrics — how success is measured
- Scope — what's in and out
- User stories / use cases — specific functional scenarios
- User journeys — step-by-step flows
- Edge cases and error states — what happens when things go wrong
- Timeline / milestones — delivery expectations

**Technical categories:**
- Stack or technology choices mentioned
- Integrations or external dependencies
- Data model or data requirements
- Performance or scalability expectations
- Security or compliance requirements
- Existing systems to integrate with
- UI framework / design mode — shadcn/ui, Radix, MUI, custom design system, or not mentioned (used to decide `designMode` in PROJECT_PROFILE.md)

**Organizational categories:**
- Team or roles mentioned
- Constraints (budget, time, team skills)
- Dependencies on other teams or projects
- Stakeholders or approvers

### Step 3: Classify every element

For each piece of information found, classify it as:

- ✅ **Clear** — specific, complete, unambiguous, actionable as-is
- ⚠️ **Ambiguous** — present but vague, contradictory, or incomplete enough to cause confusion
- ❌ **Missing** — a key decision area with no coverage at all in the document

Examples:

| Element | Status | Reason |
|---------|--------|--------|
| "The app should be fast" | ⚠️ Ambiguous | "Fast" is undefined — no latency targets, no context |
| "Users must be authenticated" | ✅ Clear | Unambiguous requirement |
| "Success metrics" | ❌ Missing | Not mentioned anywhere |
| "Supports mobile and desktop" | ⚠️ Ambiguous | No breakpoints, no native vs. responsive specified |
| "Built with shadcn/ui" | ✅ Clear | UI framework set → designMode = shadcn-theme |
| No UI framework mentioned | ❌ Missing | Critical gap — designMode must be resolved before design-flow |

### Step 4: Generate the gap list

Produce only the questions that actually need asking. Each question should be:
- Targeted — based on a specific gap or ambiguity found in the document
- Specific — "You mentioned 'users' — do you mean end consumers, admin users, or both?" not "Tell me about your users"
- Prioritized — critical gaps first, nice-to-haves last

Group questions by priority:

**Critical gaps** — decisions that block architecture or workstream definition
**Important gaps** — decisions that affect scope or user story completeness
**Minor gaps** — clarifications that would improve quality but aren't blockers

### Step 5: Write the intake document

Save to `.pm/<feature-name>/INTAKE.md`:

```markdown
# Intake Analysis: [Feature/Project Name]

**Source document:** [filename or description]
**Analyzed:** [date]

---

## What's already clear ✅

### Problem and users
- [Element]: [what was stated, verbatim or summarized]
- ...

### Scope
- [Element]: [what was stated]
- ...

### Technical
- [Element]: [what was stated]
- ...

### Organizational
- [Element]: [what was stated]
- ...

---

## What needs clarification ⚠️

| # | Element | What was said | What's unclear | Priority |
|---|---------|--------------|----------------|----------|
| A1 | [element] | "[quote from doc]" | [specific ambiguity] | Critical |
| A2 | [element] | "[quote]" | [ambiguity] | Important |
...

---

## What's missing entirely ❌

| # | Category | Why it matters | Priority |
|---|----------|---------------|----------|
| M1 | Success metrics | Without these, we can't validate the project worked | Critical |
| M2 | Error states | Needed for UI design and API error handling | Important |
...

---

## Gap summary

| Status | Count |
|--------|-------|
| ✅ Clear | [N] items |
| ⚠️ Ambiguous | [N] items |
| ❌ Missing | [N] items |

**Readiness score:** [Low / Medium / High]
- Low: Many critical gaps — the document is a starting point, not a spec
- Medium: Core is clear but important details are missing
- High: Mostly complete — a short grill will finalize it

---

## Grill question list (for pm-grill)

### Critical — must resolve before architecture

1. [Specific question targeting gap A1 or M1]
2. [Specific question]
...

### Important — should resolve before tasks

1. [Specific question]
...

### Minor — nice to have

1. [Specific question]
...
```

### Step 6: Hand off to grill

After saving, summarize to the user:

> "I found [N] clear elements, [N] ambiguities, and [N] missing areas. The most critical gaps are: [top 3 items]. Ready to move to the grill and resolve these?"

Pass the INTAKE.md gap list to `pm-grill` — the grill should only ask the questions in the "Grill question list" section, in priority order.

## Rules

- Don't discard anything from the original document — if the user wrote it, it's intentional until proven otherwise
- A quote should be preserved verbatim in the ambiguous table so the user can see exactly what triggered the question
- Never generate questions about things that are already clear — that's wasted grill time
- The readiness score is an honest assessment — if the document is half a page with no scope, say Low
- If the document is very complete (readiness High), the grill may be very short — that's fine
- If the document contains contradictions, flag them explicitly: "You stated X on page 1 and Y on page 3 — which is correct?"
