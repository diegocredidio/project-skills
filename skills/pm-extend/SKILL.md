---
name: pm-extend
description: Bootstrap a child feature slug as an explicit evolution of an existing parent slug. Writes `.pm/<child>/PARENT.md` (lineage ledger), copies `.pm/<parent>/PROJECT_PROFILE.md` (inherited or overridden), and seeds `.pm/<child>/INTAKE.md` with a Parent header + improvement summary + gap list derived from parent artifacts. Then hands off to pm-flow, which detects PARENT.md and auto-selects Path B (gap-fill) mode. Use when user says "extend <feature>", "evolve <feature>", "add <improvement> to <feature>", "melhoria em <feature>", or when pm-flow's Step 1 evolution branch invokes it. Not for greenfield planning (use pm-flow with a new slug) or for generic cross-feature discussions.
---

# PM Extend — Evolution Bootstrapper

Materialize a child slug that inherits from a parent slug. Single responsibility: write the lineage baseline and hand off to pm-flow.

## How to run

### Step 1: Collect inputs

Gather three values — ask one at a time if missing:

1. **Parent slug** — must exist at `.pm/<parent>/`. If user didn't supply, ask: "Which existing feature are you evolving? Available: [list of `.pm/*` subdirectories]."
2. **Child slug** — kebab-case name for the new feature. Offer a default derived from the user's improvement description (e.g., "add 2FA to auth" → suggest `2fa`). Ask: "Child slug `<suggested>`? Or rename."
3. **Improvement summary** — 1–5 sentence paragraph. Ask: "Describe what this evolution adds or changes vs. the parent."

### Step 2: Validate

Abort with a precise message on any of:

- Parent does not exist at `.pm/<parent>/` → "No slug `<parent>` under `.pm/`. Available: [list]. Maybe you meant pm-flow for a greenfield slug?"
- Child slug is not kebab-case → "Child slug must be kebab-case (lowercase, hyphens). Got: `<value>`."
- Child slug would collide with `.pm/<child>/` that already exists → prompt: "`.pm/<child>/` already exists. Options: [continue] (keep folder, only add/update PARENT.md + profile), [rename] (new slug), [abort]."

On `continue` with an existing folder, skip Step 4 for INTAKE.md — only write PARENT.md and PROJECT_PROFILE.md.

### Step 3: Audit parent artifacts

List what the parent folder contains. This feeds the PARENT.md artifact reference table.

Check for:
- `.pm/<parent>/PRD.md`
- `.pm/<parent>/ARCHITECTURE.md`
- `.pm/<parent>/GRILL_SUMMARY.md`
- `.pm/<parent>/PROJECT_PROFILE.md`
- `.backend/<parent>/BACKEND_BRIEF.md`, `BACKEND_STACK.md`, `BACKEND_DATA.md`, `BACKEND_API.md`
- `.frontend/<parent>/FRONTEND_BRIEF.md`, `FRONTEND_STACK.md`, `FRONTEND_ROUTES.md`
- `.design/<parent>/DESIGN_BRIEF.md`, `TOKENS.md`, `COMPONENT_SPECS.md`
- `.qa/<parent>/QA_BRIEF.md`, `QA_STRATEGY.md`, `TEST_CASES.md`

Record presence (✓) / absence (✗) for each.

If `PRD.md` and `ARCHITECTURE.md` are both absent, warn: "Parent `<parent>` has no PRD or ARCHITECTURE — the lineage will be thin. Proceed? [Y/n]"

### Step 4: Resolve profile inheritance

Read `.pm/<parent>/PROJECT_PROFILE.md`. If missing, tell the user: "Parent has no PROJECT_PROFILE.md — can't inherit. Fill it now for the parent via pm-grill Step 1.5 prompt, OR fill the child's profile from scratch?" Proceed based on choice.

If present, default to inheriting verbatim. Prompt once:

> "Parent profile: designMode=<value>, uiFramework=<value>, testingRigor=<value>. Inherit as-is? [Y/n]"

On `n`, walk each of the three fields — for each: "Keep `<parent-value>` or change to: [options]?" Record overrides + reason in PARENT.md's Profile inheritance table.

### Step 5: Write PARENT.md

Save to `.pm/<child>/PARENT.md`:

```markdown
# Parent: <parent-slug>

**Child slug:** <child-slug>
**Created:** <today ISO date>
**Improvement summary:** <the paragraph from Step 1>

## Artifacts referenced from parent

| Path | Purpose |
|------|---------|
| .pm/<parent>/PRD.md | Source FR-XXX the child extends or modifies |
| .pm/<parent>/ARCHITECTURE.md | System baseline the child builds on |
| .pm/<parent>/PROJECT_PROFILE.md | Profile inherited into child |
| .pm/<parent>/GRILL_SUMMARY.md | Decisions already resolved (skip in gap-fill) |
| .backend/<parent>/BACKEND_API.md | API surface the child may extend (if present) |
| .backend/<parent>/BACKEND_DATA.md | Data model the child may extend (if present) |
| .frontend/<parent>/FRONTEND_ROUTES.md | Route map the child may extend (if present) |
| .design/<parent>/COMPONENT_SPECS.md | Component inventory the child may reuse (if present) |

Include only the rows where the parent actually has the file. Drop rows for missing artifacts.

## Profile inheritance

| Field | Parent value | Child value | Override? |
|-------|--------------|-------------|-----------|
| designMode | <value> | <value> | no / yes — <reason> |
| uiFramework | <value> | <value> | no / yes — <reason> |
| testingRigor | <value> | <value> | no / yes — <reason> |
```

### Step 6: Copy / override PROJECT_PROFILE.md

Write `.pm/<child>/PROJECT_PROFILE.md` with the resolved values (inherited or overridden). Format matches pm-grill's canonical output:

```markdown
# Project Profile
designMode: <value>
uiFramework: <value>
testingRigor: <value>
```

### Step 7: Seed INTAKE.md

Save to `.pm/<child>/INTAKE.md`:

```markdown
# Intake Analysis: <child-slug>

**Source:** Evolution of `<parent-slug>` — see `PARENT.md`
**Analyzed:** <today>
**Mode:** evolution (lineage-aware)

## Improvement summary
<paragraph from Step 1>

## Inherited from parent (✅ clear — do NOT re-ask)

Pull the following verbatim from parent artifacts and list here so pm-grill can skip them:

- Problem domain: <from .pm/<parent>/PRD.md "Problem Statement" section — first paragraph>
- Target users: <from .pm/<parent>/GRILL_SUMMARY.md "Target Users" section>
- Stack: <from .pm/<parent>/ARCHITECTURE.md "Stack Decisions" table — one-line summary>
- Profile: designMode=<value>, uiFramework=<value>, testingRigor=<value>

If a source file is missing, write "(parent has no <section>)" instead of inventing content.

## What needs clarification for this evolution ⚠️

Derive these questions by reading the improvement summary and comparing to parent artifacts. Examples of typical questions:

| # | Element | Why it matters for the child | Priority |
|---|---------|------------------------------|----------|
| A1 | Scope boundary | Does this add new FR-XXX or modify existing FR-YYY in parent? | Critical |
| A2 | Data model delta | New entities? New columns on existing entities? Migration strategy? | Important |
| A3 | API contract impact | New endpoints only, or does this change existing endpoints? | Important |

## What's missing entirely ❌

| # | Category | Why it matters | Priority |
|---|----------|---------------|----------|
| M1 | Success metric for the evolution | Parent's metrics may not apply to the improvement alone | Critical |

## Grill question list (for pm-grill)

### Critical — must resolve before architecture
1. <derived from A1 + M1>
2. <...>

### Important — should resolve before tasks
1. <derived from A2 / A3>

### Minor
(none unless the improvement summary is notably vague)
```

Note: pm-extend is NOT pm-intake. This seeded INTAKE is deliberately thin — pm-grill (Path B) does the actual interrogation.

### Step 8: Hand off to pm-flow

Announce:

> "Child slug `<child>` bootstrapped as evolution of `<parent>`. Files written:
> - .pm/<child>/PARENT.md (lineage ledger)
> - .pm/<child>/PROJECT_PROFILE.md (inherited from parent)
> - .pm/<child>/INTAKE.md (seeded with improvement + gap list)
>
> Handing off to pm-flow. It will detect PARENT.md, auto-select Path B (gap-fill), and run pm-grill asking only evolution-specific questions."

Invoke `pm-flow` with the child slug. Do not restart the grill here; pm-flow's Step 2 takes it from here.

## Rules

- Never overwrite `.pm/<child>/INTAKE.md` if the child folder already existed before this invocation (continue-mode) — only PARENT.md and PROJECT_PROFILE.md are safe to refresh.
- Never read parent code or parent PR history — only the `.pm/<parent>/` + discipline folder artifacts. This skill is spec-aware, not code-aware.
- Never invent parent artifacts. If `.pm/<parent>/PRD.md` is missing, leave the corresponding "Inherited from parent" bullet blank and flag the gap.
- FR-IDs in the child's eventual PRD continue from `max(parent FR-XXX) + 1` — but that's pm-prd's job, not this skill's. Just note it in PARENT.md for pm-prd to read.
- Lineage is flat. If the user tries to extend an already-extended slug, allow it mechanically (parent has its own PARENT.md, which this skill ignores), but do NOT chain references. Document multi-level gap in Decision 9 of the design doc.
- Profile changes (override mode) are recorded with a reason in PARENT.md. Silent overrides are a footgun.
- This skill always ends by invoking pm-flow. Do not produce a standalone "done" state — the flow continues.
