# pm-extend — Evolution Bootstrapper — Design Doc

**Date:** 2026-04-22
**Status:** Approved, ready for implementation planning
**Scope:** A new PM skill (`pm-extend`) that bootstraps a "child" feature slug as an explicit evolution of an existing "parent" feature, inheriting `PROJECT_PROFILE.md`, seeding `INTAKE.md` with a `Parent:` header, and writing a dedicated `PARENT.md` lineage ledger. `pm-extend` does not orchestrate the flow — it prepares the folder and hands off to `pm-flow`, which detects the lineage marker and runs in Path B (gap-fill) mode. Five existing PM/discipline skills gain lineage-aware reads.

---

## Motivation

The pack's current model assumes every feature is a fresh slug: `pm-flow` Step 1 asks "what feature?" and creates `.pm/<feature>/` from scratch. This works for greenfield planning, but breaks down for the most common reality on a live project: **a new improvement lands on top of an already-shipped feature** (e.g., "add 2FA to the auth we built three months ago"; "export-to-PDF for the existing report module").

Today, three user-visible pains emerge:

1. **Lost rastreability.** The user either edits `.pm/auth/PRD.md` in place (destroying the v1 scope record) or creates `.pm/2fa/` with no link back to `auth` (downstream skills don't know the context).
2. **Repeated profile grilling.** `testingRigor`, `designMode`, and `uiFramework` were already resolved in the parent slug's `PROJECT_PROFILE.md`, yet running `pm-flow` on the child asks them again.
3. **Duplicated brief context.** Brief-intakes (`backend-brief-intake`, `frontend-brief-intake`, `design-brief-intake`, `qa-brief-intake`) audit the codebase but do not read prior-slug artifacts — so `.backend/auth/BACKEND_API.md` gets re-derived from code instead of consulted as a decision record.

The fix: a dedicated bootstrap skill that materializes the lineage on disk (`PARENT.md` + inherited `PROJECT_PROFILE.md` + `INTAKE.md` with `Parent:` header), and a handful of targeted edits in other skills to honor that marker when present. `pm-flow` and the brief-intakes already have the right shape (Path B gap-fill, codebase audit) — they just need to know when a parent exists and what artifacts to pull.

This is the public-pack version of "B" from the brainstorming conversation. Consistency-checking between parent and child (drift detection) and multi-level lineage chains are explicitly out of v1 scope; see Decision 9.

---

## Decision 1 — Skill list: one new skill

A single new skill, `pm-extend`, positioned as the **entry point for evolutions**. It does NOT orchestrate — its sole output is a prepared child slug folder. After writing the folder, it invokes `pm-flow`, which detects `PARENT.md` and proceeds in Path B (gap-fill) mode.

**Not introduced:**

- `pm-extend-flow` (orchestrator) — would duplicate `pm-flow`'s Path B logic.
- Changes to `pm-flow`'s sequence — the existing Path B sequence is already correct for evolutions; only the trigger and a couple of substeps need to become lineage-aware.

**Rationale:** the minimum skill surface that fills the gap. pm-flow already handles Path B (intake exists → gap-fill grill → PRD → architecture → ... → handoff). pm-extend writes the intake and the lineage marker; pm-flow runs unchanged from Step 2 onward.

**Skill count:** 43 → 44.

---

## Decision 2 — Lineage marker: `PARENT.md` as the canonical signal

Every lineage-aware skill keys off a single file: `.pm/<child>/PARENT.md`. Presence of that file means "this slug is an evolution of another." Absence means "this slug is standalone" (current behavior unchanged).

**Shape of `PARENT.md`:**

```markdown
# Parent: <parent-slug>

**Child slug:** <child-slug>
**Created:** 2026-04-22
**Improvement summary:** One paragraph describing what this child adds/changes vs. the parent.

## Artifacts referenced from parent

| Path | Purpose |
|------|---------|
| .pm/<parent>/PRD.md | Source FR-XXX the child extends or modifies |
| .pm/<parent>/ARCHITECTURE.md | System baseline the child builds on |
| .pm/<parent>/PROJECT_PROFILE.md | Profile inherited into child |
| .backend/<parent>/BACKEND_API.md | API surface the child may extend (if present) |
| .backend/<parent>/BACKEND_DATA.md | Data model the child may extend (if present) |
| .frontend/<parent>/FRONTEND_ROUTES.md | Route map the child may extend (if present) |
| .design/<parent>/COMPONENT_SPECS.md | Component inventory the child may reuse (if present) |

## Profile inheritance

| Field | Parent value | Child value | Override? |
|-------|--------------|-------------|-----------|
| designMode | <value> | <value> | no / yes + reason |
| uiFramework | <value> | <value> | no / yes + reason |
| testingRigor | <value> | <value> | no / yes + reason |
```

**Why a dedicated file and not just a header in INTAKE.md:** three reasons.

1. `INTAKE.md` gets overwritten by `pm-intake` if re-run; lineage info must survive re-runs.
2. Downstream skills can grep for `PARENT.md` existence cheaply without parsing arbitrary headers.
3. The artifact reference table is a ledger — it records what the child *did* consult at creation time, not what's currently in the parent folder. Useful for audit when the parent changes.

**Lineage is single-parent, flat (v1).** No chains. If the user needs to evolve `2fa` further into `2fa-sms`, they still treat `auth` as the root parent (or accept that the link to `2fa` lives in documentation, not in the marker). Multi-parent or deep chains are Decision 9 (out of scope).

---

## Decision 3 — Inputs and invocation

**Inputs (pm-extend):**

- `parent` — slug that must exist at `.pm/<parent>/`. Required.
- `name` — child slug in kebab-case. Required. Validated against collision with existing `.pm/*`.
- `improvement` — one-paragraph description of what the child adds. Required; short (1–5 sentences).

**Invocation paths:**

1. **Direct.** User says "add 2fa to auth", "melhoria em auth", "extend auth with X", or directly invokes `pm-extend`. Skill prompts for the three inputs in order if not supplied.
2. **Via `pm-flow` Step 1.** When the user runs `pm-flow` and types a name, `pm-flow` scans `.pm/` for existing slugs. If any exist, it asks: "Existing features: [auth, dashboard, ...]. Is this a new feature or an evolution of one of those?" On "evolution", `pm-flow` hands off to `pm-extend`, collects `parent` + `improvement`, then resumes Step 2 of its own flow (which will now read `PARENT.md` and run Path B).

**Collision handling:**

- Parent not found → abort with "No slug `<parent>` under `.pm/`. Available: [list]."
- Child slug collides with existing `.pm/<name>/` → offer: `continue` (keep existing folder, only add/update `PARENT.md` + inherit profile) / `rename` (prompt new name) / `abort`.
- Parent has no `PRD.md` → warn ("parent is incomplete — artifact reference table will be thin") and proceed with whatever exists.

---

## Decision 4 — Outputs (pm-extend only)

`pm-extend` writes exactly three files into `.pm/<child>/` and then invokes `pm-flow`:

```
.pm/<child>/
  PARENT.md              # lineage marker (Decision 2 shape)
  PROJECT_PROFILE.md     # copied from parent, diffed if user overrides
  INTAKE.md              # seeded with Parent header + improvement paragraph + gap list
```

**`INTAKE.md` shape (pm-extend's seeded version):**

```markdown
# Intake Analysis: <child-slug>

**Source:** Evolution of `<parent-slug>` — see `PARENT.md`
**Analyzed:** 2026-04-22
**Mode:** evolution (lineage-aware)

## Improvement summary
<the paragraph the user provided>

## Inherited from parent (✅ clear, do not re-ask)

- Problem domain: <from parent PRD>
- Target users: <from parent GRILL_SUMMARY>
- Stack: <from parent ARCHITECTURE>
- Profile: <designMode>, <uiFramework>, <testingRigor>
- ... (verbatim pulls from parent, not re-interpreted)

## What needs clarification for this evolution ⚠️

| # | Element | Why it matters for the child | Priority |
|---|---------|------------------------------|----------|
| A1 | Scope boundary | Does this evolution add FR-XXX or modify existing FR-YYY in parent? | Critical |
| A2 | Data model delta | New entities? New columns on existing entities? | Important |
| ... |

## What's missing entirely ❌

| # | Category | Why it matters | Priority |
|---|----------|---------------|----------|
| M1 | Success metric for the evolution | Parent's metrics may not apply | Critical |
| ... |

## Grill question list (for pm-grill)

### Critical
1. <derived from A1>
2. ...
```

The seeded INTAKE is deliberately thin — pm-extend is NOT pm-intake. Its job is to set up the lineage baseline; the full grill happens in `pm-grill` when `pm-flow` resumes.

**`PROJECT_PROFILE.md` inheritance:**

- Copy parent's `PROJECT_PROFILE.md` verbatim.
- If the improvement description or invocation flags indicate a different rigor ("mvp → full because this change adds billing"), prompt once: "Parent's testingRigor is `mvp`. Keep `mvp` or upgrade to `full` for this child? (Override will be recorded in PARENT.md.)"
- Overrides are recorded in `PARENT.md`'s Profile inheritance table with the stated reason.

---

## Decision 5 — Coordinated changes in existing skills

Five existing skills gain lineage-aware behavior. Each change is gated on "if `.pm/<feature>/PARENT.md` exists, do X; otherwise behave as today."

| Skill | Change | Estimated effort |
|---|---|---|
| `pm-flow` | Step 1: after asking for the feature name, scan `.pm/` for existing slugs and offer "new or evolution?" branch. On evolution, hand off to `pm-extend`. Step 2 auto-selects Path B when `PARENT.md` exists (no "do you have existing material?" prompt). | Low |
| `pm-grill` | In gap-fill mode, when `PARENT.md` exists, read `.pm/<parent>/GRILL_SUMMARY.md` and mark inherited items as ✅ — never re-ask them. The gap list becomes "child-specific questions only." | Medium |
| `pm-prd` | When `PARENT.md` exists, add an `## Extends` section at the top of the generated PRD referencing the parent PRD's relevant FR-XXX. FR numbering in the child continues from max(parent FR-IDs)+1 to avoid cross-slug collision. | Low |
| `pm-architecture` | When `PARENT.md` exists, read `.pm/<parent>/ARCHITECTURE.md` before exploring the codebase. The "Existing stack" section in the child's architecture doc explicitly cites the parent's choices and marks deltas (new endpoints, new tables) vs. additions to existing subsystems. | Medium |
| 4× `*-brief-intake` (backend/frontend/design/qa) | When `.pm/<feature>/PARENT.md` exists, *also* read the parent's discipline artifacts (e.g., `.backend/<parent>/BACKEND_STACK.md`, `.backend/<parent>/BACKEND_API.md`) as additional context, classify them as ✅ clear inheritance (not re-asked), and record them in the intake's "Inherited from parent" section. | Medium-high (4 files) |

**Not changed (intentionally):**

- `pm-workstreams`, `pm-tasks`, `pm-review`, `pm-handoff` — their inputs are the child's own PRD/ARCHITECTURE, which already encode the lineage via Decision 5's edits to pm-prd and pm-architecture. No further wiring needed.
- Discipline-specific artifact-producing skills (`backend-stack`, `backend-api`, `frontend-stack`, `frontend-routing`, `design-tokens`, `design-components`, `qa-strategy`, `qa-cases`). They read brief-intake output, which now includes inherited context. No direct parent-reading wired into them.
- `cc-flow` / `cc-sync` — per-feature agents are rendered per child slug as today. No lineage awareness in generated agents in v1 (could be added in v1.x if agents need to know "you are the 2fa engineer, extending auth").
- `kanban-*` — epic/sub-task linking across slugs is Decision 9 (out of scope).

---

## Decision 6 — Profile inheritance semantics

**Default:** child inherits parent's `PROJECT_PROFILE.md` byte-for-byte. The child's file is a copy, not a symlink — divergence must be explicit.

**Override:** if the improvement genuinely changes the profile (rare but real — e.g., the evolution adds a feature that needs `full` rigor even though parent was `mvp`), `pm-extend` prompts once:

> "Parent's profile is designMode=shadcn-theme, uiFramework=shadcn/ui, testingRigor=mvp. Inherit as-is? (Y/n)"

On `n`, walks each field and asks: keep or change? Records the override + reason in `PARENT.md`'s Profile inheritance table.

**No cascade:** later changes to parent's `PROJECT_PROFILE.md` do NOT propagate to children automatically. The copy is point-in-time. If the user bumps parent's rigor after the child already exists, they're on their own to update the child (or accept divergence).

---

## Decision 7 — Edge cases

| Case | Behavior |
|------|----------|
| Parent slug does not exist | Abort with message listing available slugs under `.pm/`. |
| Child slug collides with existing folder | Offer continue / rename / abort. On continue, update only `PARENT.md` + `PROJECT_PROFILE.md`; leave `INTAKE.md` untouched if already present. |
| Parent is incomplete (no PRD, no ARCHITECTURE) | Warn, list what's missing, proceed with a thinner artifact reference table. The child can still be created — some inheritance is better than none. |
| Parent `PROJECT_PROFILE.md` is missing | Prompt user to fill parent's profile first via `pm-grill` step 1.5 prompt, OR fill child's profile from scratch and skip inheritance. User chooses. |
| User runs `pm-extend` on same child twice | Idempotent. Detects existing `PARENT.md`, offers: refresh (re-copy profile, re-read parent artifacts) / keep (no-op) / abort. |
| User runs `pm-flow` directly on a slug that has `PARENT.md` | pm-flow detects the marker and auto-selects Path B; no prompt for "existing material?" — the existing material *is* the lineage. |
| User deletes `PARENT.md` mid-project | Child reverts to standalone behavior on next invocation. Considered user-intent (they want to cut the link). No warning — that's their prerogative. |
| Parent slug is renamed | `PARENT.md` goes stale. No auto-detection. Documented as a manual-edit concern. |
| Improvement description is vague | pm-extend does not interrogate — it seeds the INTAKE with whatever was given and lets `pm-grill` (Path B) stress-test the gaps. |

---

## Decision 8 — User-facing surface changes

**Direct invocation flow (example):**

```
User: "adicionar 2FA ao auth"
pm-extend:
  → Inferred parent: "auth". Confirm? [Y/n]
  → Child slug: "2fa"? (auto-kebab-case, editable) [Y/n/rename]
  → Improvement summary:
     "Adicionar segundo fator de autenticação via TOTP + códigos de backup
     ao fluxo de login existente. Reutiliza sessão e UI do auth; adiciona
     tela de setup + verificação + recuperação."
  → Parent profile: designMode=shadcn-theme, uiFramework=shadcn/ui,
     testingRigor=mvp. Inherit as-is? [Y/n]
  → Writing .pm/2fa/PARENT.md, .pm/2fa/PROJECT_PROFILE.md, .pm/2fa/INTAKE.md...
  → Done. Handing off to pm-flow (Path B: gap-fill).

pm-flow:
  Step 1: .pm/2fa/ detected with PARENT.md → Path B auto-selected.
  Step 2: pm-grill in gap-fill mode, reading inherited context from
          .pm/auth/GRILL_SUMMARY.md. Asking only child-specific questions.
  ...
```

**pm-flow Step 1 update (evolution prompt):**

```
pm-flow:
  Existing features under .pm/:
    - auth  (PRD ✓, ARCHITECTURE ✓, handed off)
    - dashboard  (PRD ✓, no handoff yet)

  Is this a new feature or an evolution of one of those?
  [new / evolution of auth / evolution of dashboard / cancel]
```

On `evolution of <parent>`, pm-flow invokes `pm-extend` with `parent=<parent>`, then resumes its own Step 2 once pm-extend returns.

---

## Decision 9 — Out of scope (explicit, not forgotten)

- **Consistency check between parent and child.** pm-review does NOT flag when the child's ARCHITECTURE contradicts the parent's ARCHITECTURE. This is the "C" option from the brainstorming conversation and is deferred until we have real usage data showing which drift patterns matter. Can be added in v1.x as `pm-lineage-check` or as a mode of `pm-review`.
- **Multi-level chains (evolution of evolution).** v1 assumes single-parent. `pm-extend auth-2fa-sms parent=2fa` works mechanically but the grandparent `auth` is not auto-referenced. Documented as a known limitation.
- **Cross-feature epic linking in kanban-sync.** When `PARENT.md` exists, the child's kanban-sync could link the generated Jira epic as a sub-epic of the parent's epic. Deferred — kanban-sync's contract stays frozen for now.
- **Lineage-aware cc-sync agents.** Generated agents in `.claude/agents/<child>/` do not know they are "extending" another feature's agents. Could render a partial in v1.x.
- **Profile cascade.** Changes to parent's `PROJECT_PROFILE.md` after child exists do not propagate. Intentional — see Decision 6.
- **Automatic detection of "this PRD edit is adding a new feature, should be a child slug."** Would require diff-ing PRDs, too heuristic, belongs to human judgment. Covered by pm-flow's Step 1 prompt instead.
- **Renaming a parent slug.** Users who rename `.pm/auth/` to `.pm/authentication/` need to update `PARENT.md` in children by hand. Low-frequency operation, not worth automation in v1.

---

## Decision 10 — Rollout and risk mitigations

**Rollout:**

- Ship as v1.3.0. No breaking changes; `pm-flow` without lineage still behaves identically (evolution prompt only appears when `.pm/` already has at least one slug).
- No migration required for v1.2.x users. Projects in flight continue to work; evolution support is additive.

**Mitigations (built into design):**

- **User runs pm-extend when they meant pm-flow:** parent-existence check catches this — if parent doesn't exist, abort with a pointer to pm-flow.
- **Silent overwrites of child folder:** collision handling (Decision 7) always prompts before touching an existing `.pm/<child>/`.
- **Inherited profile becomes stale:** Decision 6 — no cascade by design. Users who want sync must update manually and knowingly.
- **Parent artifacts are missing or thin:** warnings in Decision 7 + artifact reference table in `PARENT.md` records what was *available* at creation, so downstream skills can see what was consulted vs. what was absent.
- **Child's FR-IDs collide with parent's:** pm-prd continues numbering from max(parent)+1, documented in the generated PRD header.
- **Brief-intakes double-read the codebase and the parent's artifacts, producing noise:** Decision 5 — intakes classify parent artifacts as inherited (✅ clear) and codebase findings as discovery, not merged into one pile. Noise is bounded.

---

## Summary of files changed

| File | Change |
|---|---|
| `skills/pm-extend/SKILL.md` | **New** — bootstrap skill |
| `skills/pm-flow/SKILL.md` | Edit — Step 1 evolution prompt, Step 2 auto-Path-B when PARENT.md exists |
| `skills/pm-grill/SKILL.md` | Edit — gap-fill mode reads parent GRILL_SUMMARY when PARENT.md present; inherited items marked ✅ |
| `skills/pm-prd/SKILL.md` | Edit — `## Extends` section in output + FR-ID continuation rule |
| `skills/pm-architecture/SKILL.md` | Edit — read parent ARCHITECTURE.md as context; mark deltas vs. additions |
| `skills/backend-brief-intake/SKILL.md` | Edit — read `.backend/<parent>/` artifacts when `.pm/<child>/PARENT.md` exists |
| `skills/frontend-brief-intake/SKILL.md` | Edit — same pattern for `.frontend/<parent>/` |
| `skills/design-brief-intake/SKILL.md` | Edit — same pattern for `.design/<parent>/` |
| `skills/qa-brief-intake/SKILL.md` | Edit — same pattern for `.qa/<parent>/` |
| `README.md` | Edit — skill count 43 → 44; new "Evolving a feature" section; pm-extend entry in flow map |
| `docs/skills-flow.md` | Edit — lineage branch in mermaid; `PARENT.md` reader map row |
| `.claude-plugin/plugin.json` | Edit — version 1.2.0 → 1.3.0; description mentions "lineage-aware evolutions" |
| `.claude-plugin/marketplace.json` | Edit — mirror |
| `CHANGELOG.md` | Edit — new `[1.3.0]` entry |
| `package.json` | Edit — version bump |
| `docs/plans/2026-04-22-pm-extend-design.md` | This doc |
| `docs/plans/2026-04-22-pm-extend.md` | Implementation plan (next — via writing-plans) |

**New runtime artifacts (in consumer projects, not this repo):** `.pm/<child>/PARENT.md` for each evolved slug.

---

## Open questions (deferred)

- **When does `pm-flow`'s Step 1 evolution prompt become noisy?** On a project with 20 slugs, listing them all inline may clutter. v1 lists them; v1.x may paginate or fuzzy-match.
- **Should `pm-extend` support batch mode** (e.g., "evolve auth with these 3 improvements as separate children")? Unclear demand; out of v1.
- **Does the `PARENT.md` artifact reference table need to be machine-checked** (fail loudly when a referenced parent artifact is deleted)? v1 is informational only; v1.x could add a `pm-lineage-validate` skill.
- **Should `cc-sync` rendered agents include a "you are evolving <parent>, see <parent> agents for baseline" line?** Likely yes once we see dev workflows; holding for v1.x based on feedback.
