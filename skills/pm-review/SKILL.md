---
name: pm-review
description: Reads prior PM artifacts from `.pm/<feature>/` and produces `.pm/<feature>/REVIEW.md` — audits the plan for gaps, risks, missing dependencies, incomplete coverage (user story → task, API → consumer, design → implementation). Use when user says "review the plan", "find gaps", "what am I missing", "audit the tasks", or when the pm-flow orchestrator triggers this step. Not for code review, PR review, or reviewing non-PM artifacts.
---

# PM Review — Plan Review and Gap Analysis

Run a structured review of all PM artifacts to find what's missing, what's risky, and what doesn't add up.

## How to run

### Step 1: Load all artifacts

Read every file in `.pm/<feature-name>/`:
- GRILL_SUMMARY.md
- PRD.md
- ARCHITECTURE.md
- WORKSTREAMS.md
- TASKS.md

If any are missing, note it — that's already a finding.

### Step 2: Run the review checklist

#### Coverage check — User stories → Tasks
For every user story in the PRD:
- Does at least one task address it?
- Are the acceptance criteria in the task consistent with the user story?
- Are edge cases from the user story reflected in task acceptance criteria?

Flag: "User story US-X has no corresponding task" or "User story US-X edge case Y is not covered in any task"

#### Coverage check — Functional requirements → Architecture → Tasks
For every FR-XXX in the PRD:
- Is there a corresponding API endpoint or system component in ARCHITECTURE.md?
- Is there a corresponding task in TASKS.md?

Flag: "FR-XXX is defined but has no API endpoint" or "FR-XXX has an endpoint but no implementation task"

#### Dependency check
For every task with dependencies:
- Does the dependency actually exist?
- Is the dependency in an earlier phase?
- Are there circular dependencies?
- Are there implicit dependencies not listed? (e.g., task uses an API that's defined in another task)

Flag: "TASK-X depends on TASK-Y but TASK-Y is in a later phase" or "TASK-X uses endpoint /api/foo but no task creates that endpoint"

#### API contract check
For every API endpoint in ARCHITECTURE.md:
- Is there a backend task to implement it?
- Is there a frontend task to consume it?
- Are the request/response shapes consistent between the architecture doc and the task descriptions?

Flag: "Endpoint POST /api/users is defined but no frontend task consumes it"

#### Workstream balance check
- Is one workstream disproportionately loaded?
- Are there phases where one workstream has nothing to do?
- Are the parallel work opportunities realistic?

Flag: "Backend has 15 tasks in Phase 2 but Frontend has 2 — this suggests a bottleneck"

#### Design coverage check
- Does every user-facing page have a design task?
- Are empty states designed?
- Are error states designed?
- Are loading states designed?
- Is responsive behavior addressed?
- Is accessibility addressed?

Flag: "Page /dashboard has no design task" or "No task addresses empty state for [feature]"

#### Risk identification
Look for:
- Single points of failure (one integration everything depends on)
- Unproven technology choices
- Missing error handling
- No monitoring or observability
- No rollback strategy
- Missing data migration strategy (if applicable)
- Performance risks (N+1 queries, large payloads, unbounded lists)
- Security gaps (missing auth on endpoints, no input validation)

#### Open questions check
- Are there still unresolved questions from the grill summary?
- Do any tasks have ambiguous acceptance criteria?
- Are there TODO or TBD markers in any document?

### Step 3: Write the review document

```markdown
# Plan Review: [Feature/Project Name]

**Reviewed:** [Date]
**Documents reviewed:** [List all files]

---

## Summary

| Category | Issues found | Critical | Warning | Info |
|----------|-------------|----------|---------|------|
| Coverage gaps | [N] | [N] | [N] | [N] |
| Dependency issues | [N] | [N] | [N] | [N] |
| Design gaps | [N] | [N] | [N] | [N] |
| Technical risks | [N] | [N] | [N] | [N] |
| Open questions | [N] | [N] | [N] | [N] |
| **Total** | **[N]** | **[N]** | **[N]** | **[N]** |

**Overall assessment:** [Ready to start / Needs work / Major gaps]

---

## Critical Issues (must fix before starting)

### REVIEW-001: [Issue title]
- **Category:** [Coverage gap / Dependency / Risk / etc.]
- **Description:** [What's wrong]
- **Impact:** [What happens if we don't fix this]
- **Recommendation:** [Specific action to take]
- **Affects:** [TASK-XXX, FR-XXX, etc.]

---

## Warnings (should fix soon)

### REVIEW-010: [Issue title]
[Same format]

---

## Informational (nice to address)

### REVIEW-020: [Issue title]
[Same format]

---

## Coverage Matrix

| User Story | PRD FR | Architecture | Workstream | Task | Design | Status |
|-----------|--------|-------------|------------|------|--------|--------|
| US-1 | FR-001 | ✅ Endpoint | ✅ Backend | ✅ TASK-010 | ✅ TASK-004 | ✅ Covered |
| US-2 | FR-003 | ✅ Endpoint | ✅ Frontend | ❌ Missing | ❌ Missing | ⚠️ Gap |
...

---

## Recommendations

### Immediate actions
1. [Action with specific instruction]
2. [Action]

### Before starting Phase 2
1. [Action]

### Ongoing monitoring
1. [What to watch for during implementation]
```

### Step 4: Review with user

Present the findings and prioritize:
- "Here are [N] critical issues that should be fixed before starting work."
- "Here are [N] warnings you'll want to address soon."
- "Should I update the tasks/architecture/PRD to fix any of these?"

If the user approves fixes, update the relevant files and re-run the affected checks.

### Step 5: Save

Save to `.pm/<feature-name>/REVIEW.md`.

## Rules

- Be thorough but not pedantic — flag real risks, not theoretical ones
- Critical issues = "this will cause rework or failure if not addressed"
- Warnings = "this will cause confusion or inefficiency"
- Info = "this could be improved but won't block progress"
- The coverage matrix is the most important output — it shows at a glance what's covered and what's not
- If you find a gap, always include a specific recommendation, not just "fix this"
- If the plan is solid, say so — don't manufacture issues to seem thorough
