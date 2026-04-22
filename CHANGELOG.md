# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/), and this
project adheres to [Semantic Versioning](https://semver.org/).

## [1.4.0] — 2026-04-22

### Added

- `bdd-flow` skill — thin orchestrator for the BDD session flow. Verifies PRD.md + ARCHITECTURE.md are present, creates `.bdd/<feature>/`, invokes `bdd-scenarios`, and hands back with next-step guidance.
- `bdd-scenarios` skill — Three Amigos session conductor. Loads PRD + ARCHITECTURE (+ parent BDD files if lineage), scopes the session (inherited vs in-scope FRs), facilitates a per-FR interactive session with three question rounds (PM / Dev / QA perspectives), drafts backend and frontend scenarios separately, iterates until user approves, then writes `.bdd/<feature>/BACKEND_FEATURES.md` (API-level Gherkin) and `FRONTEND_FEATURES.md` (UI-level Gherkin) with coverage indexes. Lineage-aware: only runs session for new/modified FRs; inherited FRs get reference comments.

### Changed

- `pm-handoff` — new pre-discipline option "BDD session (optional, pre-discipline)" invokes `bdd-flow` before workstream selection. After the session, user returns to the normal workstream menu. Step 1 optional inputs updated to list BDD feature files.
- `backend-brief-intake` — new Step 2.6: reads `.bdd/<feature>/BACKEND_FEATURES.md` if present. FR-IDs with matching scenarios are classified as behaviorally specified (✅ clear) without grilling. Grill gate extended: never re-grill FRs covered by BDD scenarios. New Rules bullet.
- `frontend-brief-intake` — same Step 2.6 pattern, reading `.bdd/<feature>/FRONTEND_FEATURES.md`.
- `qa-cases` — new Step 1.5 mode detection: if both BDD feature files are present, switches to augment mode (imports FR anchors for coverage matrix, adds edge cases under "Additional edge cases" section, skips scenario rewriting). TEST_CASES.md header gains optional `**Scenario source:**` line. New Rules bullet. Standard mode unchanged when BDD files absent.
- README: skill count 44 → 46, eight families (added bdd-flow), new "BDD-first workflow" section, `bdd-flow?` in ASCII flow map, `.bdd/<feature>/` in artifacts table.
- `docs/skills-flow.md`: 44 → 46 in intro, `bdd-flow` node wired into PM flow mermaid with BDD files feeding backend-flow, frontend-flow, and qa-cases.

## [1.3.0] — 2026-04-22

### Added

- `pm-extend` skill — lineage-aware Step-0 entry point for evolving an existing feature. Writes `PARENT.md` (lineage ledger: parent slug, present-artifact table, FR-ID baseline), copies `PROJECT_PROFILE.md` to the child slug (inheriting `designMode`, `uiFramework`, `testingRigor`), seeds `INTAKE.md`, then hands off to `pm-flow` which auto-selects Path B.
- `PARENT.md` lineage ledger — canonical signal that activates lineage-aware mode in all downstream skills. Written by `pm-extend`; read by `pm-grill`, `pm-prd`, `pm-architecture`, `backend-brief-intake`, `frontend-brief-intake`, `design-brief-intake`, and `qa-brief-intake`.

### Changed

- `pm-flow` — Step 1 now shows evolution slugs alongside new-feature option and auto-selects Path B when `PARENT.md` is present (skipping the existing-material prompt). New Rules bullet scopes `pm-extend` responsibility boundary.
- `pm-grill` — Step A1 (gap-fill mode) adds Lineage check: reads parent `GRILL_SUMMARY.md` + `PROJECT_PROFILE.md` when `PARENT.md` exists. User-facing opening message has two templates (with / without lineage) to prevent conditional-parenthetical leakage. New Rules bullet.
- `pm-prd` — Step 1 reads `**FR-ID baseline:**` from `PARENT.md` as primary FR-ID source; reads parent `PRD.md` as secondary for related-FR identification. Adds lineage-only `## 0. Extends` section to template (HTML-comment sentinel + pinned Relationship vocabulary). New Rules bullet: child FR-IDs continue from `max(parent) + 1`.
- `pm-architecture` — Step 1 reads parent `ARCHITECTURE.md` when `PARENT.md` is present. Adds `### Lineage` subsection to §1 System Overview and Status column (`inherited / extended / new`) to §2 Stack Decisions. New Rules bullets: inherit same major versions, no silent divergence.
- `backend-brief-intake` — new Step 2.5 reads parent `BACKEND_STACK.md`, `BACKEND_DATA.md`, `BACKEND_API.md` and classifies inherited items; Step 4 grill gate excludes items classified as inherited; template gains `## Parent baseline` section; new Rules bullet locks backend conventions from parent.
- `frontend-brief-intake` — same Step 2.5 / grill-gate / template pattern as backend, covering `FRONTEND_STACK.md`, `FRONTEND_ROUTES.md`, `COMPONENT_PLAN.md`.
- `design-brief-intake` — same Step 2.5 / template pattern, covering `TOKENS.md`, `COMPONENT_SPECS.md`, `IA.md`; section heading `## Parent design baseline`.
- `qa-brief-intake` — same Step 2.5 / template pattern, covering `QA_STRATEGY.md`, `TEST_CASES.md`; section heading `## Parent QA baseline`; additive TC rule (child TCs never duplicate parent TCs).
- README: skill count 43 → 44, new "Evolving a feature" section, `pm-extend? ↘` entry in ASCII flow map, `PARENT.md` added to `.pm/<feature>/` artifacts table.
- `docs/skills-flow.md`: 43 → 44 in intro, `pm-extend` node wired into PM flow mermaid.

## [1.2.0] — 2026-04-20

### Added

- `cc-flow` and `cc-sync` skills — compile planning artifacts into per-feature Claude Code agents in `.claude/agents/<feature>/`. Setup mode copies starter templates; cycle mode renders agents idempotently via manifest + checksum-prompt policy.
- Starter templates in `templates/` (5 agent templates + 4 partials): `backend-engineer`, `frontend-engineer`, `qa-engineer`, `product-architect`, `design-reviewer`; partials `testing-rigor-mvp`, `testing-rigor-full`, `shadcn-theme-notes`, `custom-system-notes`.
- `docs/guardrails.md` — end-to-end explainer: what cc-flow generates, why, the flow, checksum-prompt policy, manifest anatomy, archive lifecycle, FAQ.
- `testingRigor`-aware and `designMode`-aware partial selection in cc-sync.

### Changed

- `pm-handoff` gains a 5th workstream option "CC Guardrails" to bootstrap Claude Code agents via `cc-flow`.
- `kanban-pickup` suggests the per-feature agent (`@<feature>-<discipline>-engineer`) when `.claude/agents/<feature>/` exists after task pickup.
- README: skill count 41 → 43, new "Claude Code guardrails" family, seven families total, cc-flow in ASCII flow diagram, new artifacts section, new "Guardrails as output" principle.
- `docs/skills-flow.md`: new CC flow mermaid diagram, cc-sync added to PROJECT_PROFILE.md reader map, CC rows added to artifacts table, 41 → 43 in intro.

## [1.0.0] — 2026-04-17

Initial release. 35 skills across five disciplines:

### Added

- **Dual-distribution packaging** — single tree works for both the native
  Claude Code plugin format (`/plugin marketplace add diegocredidio/project-skills`)
  and the Vercel Skills CLI (`npx skills add diegocredidio/project-skills`).
- **Plugin manifest** at `.claude-plugin/plugin.json` (v1.0.0).
- **Self-referencing marketplace** at `.claude-plugin/marketplace.json`.
- **9 PM skills** (migrated from the prototype into `skills/`, with edits to
  descriptions to avoid collision with `superpowers:*` and to strip
  slash-command prefixes from cross-references): `pm-flow`, `pm-intake`,
  `pm-grill`, `pm-prd`, `pm-architecture`, `pm-workstreams`, `pm-tasks`,
  `pm-review`, `pm-handoff`.
- **8 Design skills** (new): `design-flow`, `design-grill`,
  `design-brief-intake`, `design-ia`, `design-tokens`, `design-components`,
  `design-tasks`, `design-review`.
- **7 Backend skills** (replaces the monolithic `backend-flow`): `backend-flow`
  (now an orchestrator), `backend-brief-intake`, `backend-stack`, `backend-data`,
  `backend-api`, `backend-tasks`, `backend-review`.
- **7 Frontend skills** (replaces the monolithic `frontend-flow`):
  `frontend-flow` (now an orchestrator), `frontend-brief-intake`,
  `frontend-stack`, `frontend-routing`, `frontend-components`,
  `frontend-tasks`, `frontend-review`.
- **4 Kanban skills** for Jira sync via MCP (Atlassian): `kanban-flow`,
  `kanban-sync`, `kanban-status`, `kanban-pickup`.

### Changed

- `backend-flow` and `frontend-flow` decomposed from monolithic single skills
  into orchestrators + specialist sub-skills, mirroring the decomposition
  pattern already used by the PM skills.
- `pm-handoff` now invokes the in-package `design-flow` instead of pointing
  at `julianoczkowski/designer-skills`. Julian's work remains credited as
  inspiration.
- Every skill description now includes an anti-collision clause delimiting
  scope vs. `superpowers:brainstorming`, `superpowers:writing-plans`, and
  `superpowers:executing-plans`.
- Cross-skill references now use bare names (e.g. `pm-grill`) instead of
  slash-command prefixes (`/pm-grill`), so the same SKILL.md works both when
  installed as a plugin (`project-skills:pm-grill`) and as standalone
  skills (`pm-grill`).

### Removed

- Non-standard `"skills"` array from `package.json` (was never read by any
  tool).
- External dependency on `julianoczkowski/designer-skills` at runtime.
