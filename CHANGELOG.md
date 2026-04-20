# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/), and this
project adheres to [Semantic Versioning](https://semver.org/).

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
