---
name: pm-prd
description: Reads prior PM artifacts from `.pm/<feature>/GRILL_SUMMARY.md` (or user-provided context) and produces a structured Product Requirements Document at `.pm/<feature>/PRD.md`. Use when user says "write a PRD", "create a product requirements document", "document the requirements", "formalize the plan", or when the pm-flow orchestrator triggers this step. Not for free-form writing coaching or generic document drafting.
---

# PM PRD — Product Requirements Document

Transform the grilling session (or raw input) into a structured, stakeholder-ready PRD.

## How to run

### Step 1: Gather inputs

Check for existing artifacts:
1. Look for `.pm/<feature-name>/GRILL_SUMMARY.md` — if it exists, use it as the primary input
2. If no grill summary exists, ask the user for a detailed description. Consider suggesting they run `pm-grill` first.
3. Explore the codebase to understand existing patterns, APIs, components, and constraints
4. **Lineage check:** if `.pm/<feature-name>/PARENT.md` exists:
   - Read `PARENT.md`'s `**FR-ID baseline:**` line — this is the recorded max parent FR-XXX; child FR-IDs start at `baseline + 1`.
   - Read the parent slug from `PARENT.md`'s H1 (`# Parent: <parent-slug>`).
   - Read `.pm/<parent-slug>/PRD.md` to identify which parent FRs the child's improvement relates to (these populate the Extends table).
   - If the parent PRD's actual max FR-XXX exceeds the recorded baseline (baseline was stale), trust the PRD value and note the drift in the generated PRD's Open Questions section.

### Step 2: Ask PRD-specific questions

Before writing, clarify anything not covered by the grill:
- "Who is the primary audience for this PRD? Engineers? Stakeholders? Both?"
- "Are there existing PRD conventions in your team I should follow?"
- "How detailed should the user stories be?"

### Step 3: Write the PRD

Use this template. Adapt sections based on project complexity — a small feature needs less than a full product.

**Lineage-only sections:** sections marked with an HTML comment starting `<!-- lineage-only: ... -->` must be emitted ONLY when `.pm/<feature-name>/PARENT.md` exists. If lineage is absent, omit the entire section (including the comment and heading). When emitting, delete the HTML comment line — it's an authoring marker, not PRD content.

```markdown
# PRD: [Feature/Project Name]

**Author:** [User name or team]
**Date:** [Today]
**Status:** Draft
**Version:** 1.0

---

<!-- lineage-only: emit this entire section ONLY when PARENT.md exists; delete this comment on emit -->
## 0. Extends

**Parent feature:** `<parent-slug>` — see `.pm/<parent>/PRD.md`
**This PRD extends:** list the parent FR-IDs this evolution modifies or builds on.
**FR-ID continuation:** FR-XXX in this PRD start at `max(parent FR-IDs) + 1` to avoid cross-slug collision.

**Relationship vocabulary (use only these values):**
- `extends` — child FR builds on parent FR's behavior
- `modifies` — child FR changes parent FR's behavior
- `supersedes` — child FR replaces parent FR entirely
- `unchanged` — parent FR carries over untouched; Child FR column = `—`

| Parent FR | Relationship | Child FR | Summary |
|-----------|--------------|----------|---------|
| FR-[NNN]  | [extends\|modifies\|supersedes\|unchanged] | FR-[MMM] or `—` | [one-line summary] |
| FR-[NNN]  | unchanged    | —        | [parent FR carries over untouched] |

## 1. Problem Statement

[What problem exists? Who has it? How painful is it?
Write from the user's perspective, not the builder's.]

## 2. Proposed Solution

[High-level description of what we're building.
Focus on WHAT, not HOW. No implementation details.]

## 3. Goals and Success Metrics

| Goal | Metric | Target |
|------|--------|--------|
| [Primary goal] | [Measurable metric] | [Specific target] |
| [Secondary goal] | [Measurable metric] | [Specific target] |

### Non-goals
[What this project explicitly does NOT try to achieve]

## 4. User Stories

[Extensive, numbered list. Every interaction path should be covered.]

1. As a [role], I want [capability], so that [benefit]
2. ...

### Edge cases
[Numbered list of edge case scenarios and expected behavior]

## 5. Scope

### v1 — Must have
[Features required for initial release]

### v2 — Should have
[Features for fast-follow]

### Out of scope
[Explicitly excluded — with reasoning]

## 6. User Journeys

### Journey 1: [Name]
1. User [action]
2. System [response]
3. User [sees/does]
...

### Journey 2: [Name]
...

## 7. Functional Requirements

### [Area 1]
- FR-001: [Requirement description]
- FR-002: ...

### [Area 2]
- FR-010: [Requirement description]
...

## 8. Non-Functional Requirements

- **Performance:** [Response times, throughput]
- **Security:** [Auth, data protection, compliance]
- **Accessibility:** [WCAG level, specific requirements]
- **Scalability:** [Expected load, growth]
- **Reliability:** [Uptime, error budget]

## 9. Constraints and Dependencies

### Technical constraints
[Existing systems, tech stack limitations]

### External dependencies
[Third-party APIs, vendor timelines]

### Team constraints
[Skills available, capacity]

## 10. Open Questions

| # | Question | Owner | Deadline |
|---|----------|-------|----------|
| 1 | [Question] | [Who should answer] | [When] |

## 11. Appendix

### Key decisions log
[Decisions made during grilling/planning and their rationale]

### References
[Links to related docs, competitor analysis, research]
```

### Step 4: Review with user

Present the PRD and ask:
- "Does the problem statement capture the real pain?"
- "Are any user stories missing?"
- "Is the scope right for v1?"
- "Any stakeholder concerns I should address?"

Iterate until the user confirms.

### Step 5: Save

Save to `.pm/<feature-name>/PRD.md`.

## Rules

- User stories should be EXHAUSTIVE — cover every interaction path, not just the happy path
- Never include implementation details (no file paths, no code, no specific technology choices) — that's for `pm-architecture`
- Write for the broadest audience — a designer, a developer, and a business stakeholder should all understand it
- If the grill summary has open questions, flag them prominently — don't bury them
- Each functional requirement gets a unique ID (FR-001) for traceability in later steps
- When `.pm/<feature-name>/PARENT.md` exists, emit the `## 0. Extends` section at the top of the PRD and start FR numbering from `max(parent FR-IDs) + 1`. Never reuse parent FR-IDs — they are traceable identifiers across slugs.
