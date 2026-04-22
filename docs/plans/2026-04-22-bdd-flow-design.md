# bdd-flow — BDD Scenario Conductor — Design Doc

**Date:** 2026-04-22
**Status:** Approved, ready for implementation planning
**Scope:** Two new skills (`bdd-flow` orchestrator + `bdd-scenarios` Three Amigos conductor) and four targeted edits to existing skills (`pm-handoff`, `backend-brief-intake`, `frontend-brief-intake`, `qa-cases`) to support a BDD-first workflow. BDD is optional — teams that don't use it see no change in behavior.

---

## Motivation

The pack today produces Gherkin scenarios via `qa-cases`, but only after backend and frontend flows complete. Those scenarios document what was built; they do not drive what gets built. Teams that practice BDD need the opposite ordering: scenarios agreed upon **before** discipline flows start, so both backend and frontend developers implement against a shared behavioral contract.

Three pain points on BDD teams today:

1. **Wrong ordering.** `qa-cases` reads `BACKEND_API.md` and `COMPONENT_SPECS.md` — artifacts that only exist after discipline flows. Scenarios therefore can't be written until after implementation decisions are locked.
2. **No shared contract.** Backend and frontend both interpret the PRD independently. There is no single behavioral specification that both discipline flows reference as the source of truth.
3. **No lineage-awareness.** When evolving a feature (via `pm-extend`), `qa-cases` does not know which parent scenarios are inherited vs. which ones are new — so it either re-derives everything or leaves the gap.

The fix: a `bdd-flow` that runs after `pm-architecture` and before discipline flows begin, producing two discipline-scoped feature files. Those files become first-class inputs to `backend-brief-intake`, `frontend-brief-intake`, and `qa-cases`.

---

## Decision 1 — Skill count: two new skills

Two new skills:

- `bdd-flow` — thin orchestrator. Verifies inputs, invokes `bdd-scenarios`, then hands back to user.
- `bdd-scenarios` — Three Amigos conductor. Loads PRD + ARCHITECTURE (+ parent features if lineage), facilitates a per-FR interactive session, produces two output files.

**Not introduced:**
- `bdd-brief` (separate input-loading skill) — too thin to justify its own skill. Input loading is Step 1 of `bdd-scenarios`.
- Separate `bdd-backend-scenarios` + `bdd-frontend-scenarios` — the Three Amigos session covers both disciplines in a single conversation; splitting into two skills would force the user to run the session twice.

**Skill count:** 44 → 46.

---

## Decision 2 — Position in the flow

`bdd-flow` is optional and runs **after `pm-architecture`, before discipline flows**. It is invoked:

- From `pm-handoff` as a new pre-discipline option ("Run BDD session first → then start discipline flows")
- Or directly by the user at any point after `pm-architecture` is written

`pm-handoff` today asks which workstreams to start. The new option reads:

> "BDD session (optional) → generates behavioral contracts before discipline flows start"

If the user picks BDD session: `pm-handoff` invokes `bdd-flow`, then continues with discipline flow selection.
If the user skips it: behavior unchanged — discipline flows run without BDD contracts.

---

## Decision 3 — Output artifacts and folder

Two files, in a dedicated `.bdd/<feature>/` folder:

```
.bdd/<feature>/
  ├── BACKEND_FEATURES.md    (API-level Gherkin scenarios)
  └── FRONTEND_FEATURES.md   (UI/interaction-level Gherkin scenarios)
```

**Why `.bdd/` not `.backend/` or `.frontend/`?**
The BDD session is a cross-discipline artifact produced by the Three Amigos (PM + Dev + QA) — not by a discipline flow. Placing it inside `.backend/` or `.frontend/` would imply it was written by that flow, which it wasn't. A dedicated `.bdd/` folder makes the source clear.

**Naming rationale:**
- `BACKEND_FEATURES.md` — API-level scenarios; consumed by `backend-brief-intake`
- `FRONTEND_FEATURES.md` — UI/interaction-level scenarios; consumed by `frontend-brief-intake`
- Both consumed by `qa-cases` for coverage matrix

---

## Decision 4 — Three Amigos session model: skill conducts

`bdd-scenarios` conducts the session interactively. For each FR (or FR cluster when journeys span multiple):

1. Present the FR title + description from PRD
2. Ask three rounds of questions:
   - **PM lens:** "What is the exact business rule? What must NOT happen?"
   - **Dev lens:** "What is the expected API contract / UI interaction? Technical constraints?"
   - **QA lens:** "What is the critical negative path? What edge case always gets forgotten?"
3. Draft scenarios from answers (backend-scoped and frontend-scoped separately)
4. Present drafts to user for approval/iteration
5. Confirm or iterate until user accepts
6. Move to next FR

**Follow-up on vague answers:** if any answer is ambiguous, the skill asks one clarifying follow-up before drafting. It does not guess.

**Batching:** FRs that share the same user journey are batched together in one session round to avoid repetition.

---

## Decision 5 — Scenario format: Gherkin in Markdown, discipline-scoped

Both output files use Gherkin syntax (Feature / Scenario / Given-When-Then) embedded in Markdown, organized by FR-ID.

**BACKEND_FEATURES.md** — API-level scenarios:
- Scenarios describe HTTP requests and responses (status codes, body shape, headers)
- No UI references
- Named by endpoint × state combination

**FRONTEND_FEATURES.md** — UI/interaction-level scenarios:
- Scenarios describe user interactions (clicks, form submissions, navigation)
- No direct API endpoint references (those are an implementation detail)
- Named by user action + expected outcome

Both files use FR-IDs as anchors so `qa-cases` can build a coverage matrix.

**Framework agnostic:** pure Gherkin, no Cucumber/Behave/SpecFlow boilerplate. Step definitions are written by the implementation team; the skill only writes the `.feature` content.

---

## Decision 6 — Lineage-aware behavior

When `.pm/<feature>/PARENT.md` exists (feature created via `pm-extend`):

`bdd-scenarios` Step 1 additionally reads:
- `.bdd/<parent-slug>/BACKEND_FEATURES.md` (if present)
- `.bdd/<parent-slug>/FRONTEND_FEATURES.md` (if present)

If parent features exist:
- Present them as the inherited baseline at session start
- Only run the Three Amigos session for FRs that are **new or modified** (not for FRs classified as `unchanged` in the PRD Extends table)
- In the output files, inherited scenarios are referenced with a note `<!-- inherited from <parent-slug> — do not duplicate -->` rather than copy-pasted
- New/modified scenarios are written in full

If parent features do not exist (parent was planned before bdd-flow existed):
- Treat as standalone session, write all scenarios from scratch
- Add a note in the preamble: "Parent `<parent-slug>` has no BDD artifacts — all scenarios written from scratch."

---

## Decision 7 — Downstream consumer changes

### `pm-handoff`
Add a new pre-discipline option: "BDD session → generates `.bdd/<feature>/` before discipline flows". Positioned before the workstream list. If selected, invokes `bdd-flow` then returns to workstream selection.

### `backend-brief-intake`
New Step 2.6 (after Step 2.5 lineage read, before Step 3 classify): check for `.bdd/<feature>/BACKEND_FEATURES.md`. If present, read it and list behavioral contracts as an additional intake dimension. These inform Step 3 classification: a backend concern with a matching scenario in BACKEND_FEATURES is ✅ clear (behavior is specified). Step 4 grill gate: do not re-grill concerns that have a matching scenario.

### `frontend-brief-intake`
Same pattern as backend: new Step 2.6 reads `.bdd/<feature>/FRONTEND_FEATURES.md` and informs classification.

### `qa-cases`
Step 1 (load inputs) adds: check for `.bdd/<feature>/BACKEND_FEATURES.md` + `FRONTEND_FEATURES.md`. If both present:
- Skip scenario writing (Step 4 in current skill) — scenarios already exist
- Import FR-ID anchors from both files to build coverage matrix
- Add edge cases and negative cases not already covered
- Write flakiness notes as usual

If not present: current behavior unchanged.

---

## Decision 8 — `qa-cases` backward compatibility

`qa-cases` currently writes Gherkin scenarios from scratch. That behavior is preserved for teams that do not use `bdd-flow`. The check is purely additive: if BDD feature files exist → augment mode; if not → current mode.

**No breaking change** to the existing QA flow.

---

## Decision 9 — What is out of scope for v1

- `design-brief-intake` BDD integration — design scenarios (UI states, component behaviors) are a different concern from the API/UI interaction split here. Deferred.
- `qa-strategy` awareness of BDD files — qa-strategy sets pyramid shape and tools; BDD files don't change that. Deferred.
- Scenario-to-step-definition planning skill (`bdd-step-plan`) — out of scope; step definitions are written by developers against the feature files.
- BDD regression detection on lineage (did parent scenarios break?) — out of scope; that belongs in CI, not planning.
- Multi-level lineage (grandparent scenarios) — same decision as pm-extend v1; only one level of lineage read.

---

## Decision 10 — Version bump

`1.3.0` → `1.4.0` (minor: additive new skills + optional behavior in existing skills, no breaking changes).

**Skill count:** 44 → 46 (`bdd-flow`, `bdd-scenarios`).

**Modified skills (4):** `pm-handoff`, `backend-brief-intake`, `frontend-brief-intake`, `qa-cases`.
