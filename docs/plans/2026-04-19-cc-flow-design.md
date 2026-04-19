# cc-flow — Claude Code Guardrails Sync — Design Doc

**Date:** 2026-04-19
**Status:** Approved, ready for implementation planning
**Scope:** A new skill family (`cc-flow` + `cc-sync`) that compiles the planning artifacts produced by pm/design/backend/frontend/qa flows into Claude Code agents inside the consumer project. Agents are written to `.claude/agents/<feature>/`, per-feature scoped, owned by an idempotent manifest. Starter templates ship with the pack and become consumer-editable in `.cc-templates/`.

**Source plan (approved by user):** `/Users/diegocredidio/.claude/plans/stateless-pondering-pizza.md`

---

## Motivation

The pack's 43 skills today produce 5 trees of markdown artifacts (`.pm/`, `.design/`, `.backend/`, `.frontend/`, `.qa/`) with every project decision — PRD, architecture, stack, tokens, component specs, test strategy, vertical-slice tasks. Those artifacts are passive: when the dev opens Claude Code to implement, they have to manually load the relevant context in each conversation.

The fix: compile the artifacts into **executable guardrails** inside the consumer project — per-feature agents in `.claude/agents/<feature>/` that the dev invokes directly (`@<feature>-backend-engineer`) and that come pre-loaded with stack knowledge, conventions, token names, shadcn inventory, pyramid + tools, coverage targets, and explicit "do / don't" rules.

This closes the loop: **spec → Jira (via kanban-sync) → dev session (via kanban-pickup) → specialized agent pre-loaded with project guardrails**. Without cc-sync, the last hop is manual and inconsistent across devs. With it, every task lands the same way.

Companies can ship **starter templates** (bundled inside project-skills at `templates/`) that consumers copy to `.cc-templates/` and customize. Updates to the starter surface as version-drift warnings, not forced overwrites.

---

## Decision 1 — Skill list (v1 minimum)

Two new skills, gemini to the `kanban-*` pattern:

- **`cc-flow`** — orchestrator. Two modes, setup + cycle.
  - Setup mode (first run per consumer repo): copies `templates/` from project-skills into `.cc-templates/`, writes `.pm/cc-config.md`.
  - Cycle mode: invokes `cc-sync` for a named feature with preview confirmation.
- **`cc-sync`** — idempotent guardrail generator. Reads artifact trees + templates, renders, writes to `.claude/agents/<feature>/`, maintains manifest.

**Dropped from v1 (explicit):**

- `cc-status` — collapsed into `cc-sync --preview` (same as kanban-sync's Step 3 dry-run).
- `cc-setup` as a separate skill — just a mode of `cc-flow`.
- `cc-templates` — templates are static files in `.cc-templates/`, edited directly; no skill needed.

**Rationale for minimum viable:** there's genuinely only one runtime op (sync). Extra skills duplicate kanban's 4-skill shape without new value. `kanban-status` exists because boards have live state (WIP counts, stale cards); `.claude/agents/` is static output that a preview subcommand covers.

**Skill count:** 43 → 45.

---

## Decision 2 — Agent scoping: per-feature

Confirmed with user. Agents live at `.claude/agents/<feature>/<discipline>-engineer.md` — one folder per feature, mirror of the existing `.backend/<feature>/` layout.

**Reasons:**

- No cross-feature merge conflicts (10 features → 10 independent sets).
- Consumer repo with multiple features in parallel doesn't have `backend-engineer.md` flipping definition based on which feature was last synced.
- Aligns with how the rest of the pack scopes artifacts (per-feature directories in `.pm/`, `.design/`, etc.).
- Features that ship or get archived can have their agent folders archived in parallel — clean lifecycle.

**Tradeoff accepted:** 10 features × 5 agents = 50 agent files in the consumer repo. Mitigated by the subfolder layout and an auto-generated `.claude/agents/README.md` indexing owned agents by feature (optional in v1).

**Not used:** project-level agents (`backend-engineer.md` at the top-level) or the "both" approach. Can be added in v1.x if an onboarding use case emerges.

---

## Decision 3 — Template distribution: starter in project-skills + consumer-editable

Confirmed with user, with emphasis on documenting thoroughly.

**Flow:**

1. project-skills ships a starter set at `templates/` in this repo.
2. First run of `cc-flow` (setup mode) copies `templates/` → consumer's `.cc-templates/` with a `VERSION` file.
3. Consumer / company edits `.cc-templates/` to taste. These edits persist — subsequent `cc-sync` runs read from `.cc-templates/`, not from the project-skills starter.
4. When project-skills bumps `templates/VERSION`, `cc-sync` (or `cc-flow` cycle mode) warns: "starter is v1.2 but your `.cc-templates/VERSION` is v1.0 — want to diff/update?". Does NOT auto-overwrite. Consumer decides.

**Why not a remote template source** (Approach B, separate company repo): more plumbing, auth issues for private repos, adds a network dependency to the sync. Defer to v1.1 if demand.

**Documentation required** (this is the user's explicit ask):

- `templates/README.md` — in-repo docs explaining structure, editing, VERSION convention.
- `docs/guardrails.md` — top-level consumer-facing explainer: what are Claude Code guardrails, why the pack generates them, how to customize the starter, what updates look like, how to fork for company-wide use, policy on starter updates.
- `README.md` — short section linking to `docs/guardrails.md` plus showing the family position in the flow diagram.

---

## Decision 4 — Edit policy: checksum-prompt

Confirmed with user.

Manifest stores SHA-256 of each owned file at write time. On re-sync, compare current file SHA against manifest SHA:

- **SHAs match** → file unmodified since cc-sync wrote it → overwrite with new rendered content silently.
- **SHAs differ** → dev edited by hand → prompt:
  > "File `.claude/agents/user-auth/backend-engineer.md` was modified since last sync. Keep your edits, or overwrite with re-rendered template? (k / o)"
- **File missing** (dev deleted) → skip and log; don't regenerate uninvited.

User-modified section of manifest tracks files that have diverged, so the status is observable on next `cc-sync --preview`.

**Rationale:** files are heavier-ownership than Jira fields. Kanban-sync can always-overwrite because Jira issues can't be "hand-edited" the same way. Claude Code agents live on the dev's filesystem alongside their customizations — silent clobbering is a footgun.

---

## Decision 5 — Template engine: interpolation + partials (no deep conditionals)

**Syntax:** `{{var}}` for variables, `{{> partial-name}}` for partial includes. No `{{#if}}`/`{{#each}}` logic inside templates.

**Branching lives in cc-sync**, not in templates. Example: when rendering `frontend-engineer.md.tmpl`, cc-sync reads `PROJECT_PROFILE.md.designMode`, picks either `{{> shadcn-theme-notes}}` or `{{> custom-system-notes}}` partial, and renders. Template file itself is linear.

**Why partials over conditionals:** a template with 5 nested `{{#if}}` blocks covering shadcn-theme × testingRigor × frontend-framework × auth-provider × styling-lib quickly becomes illegible. Partials decompose cleanly and keep each template file scannable.

**Tradeoff accepted:** more template files (5 main + 4+ partials), but each one is short and purpose-built.

**Variables available in templates** (extracted by cc-sync from artifacts):

- `{{feature}}` — kebab-case slug
- `{{designMode}}`, `{{uiFramework}}`, `{{testingRigor}}` — from PROJECT_PROFILE.md
- `{{stack.framework}}`, `{{stack.styling}}`, `{{stack.db}}`, etc. — from BACKEND_STACK / FRONTEND_STACK decision tables
- `{{tokens.primary}}`, `{{tokens.background}}`, `{{tokens.radius}}` — from TOKENS.md
- `{{shadcnComponents}}` — list from COMPONENT_SPECS shadcn-theme inventory
- `{{apiEndpoints}}` — list from BACKEND_API.md
- `{{routes}}` — list from FRONTEND_ROUTES.md
- `{{coverageTargets}}` — from QA_STRATEGY.md
- `{{testTools}}` — from QA_STRATEGY.md
- `{{artifactPaths}}` — precomputed paths the agent references (e.g. `/Users/.../project/.design/<feature>/TOKENS.md`)

---

## Decision 6 — Manifest format + lifecycle

Path: `.claude/.cc-manifest.md`. Markdown table format (consistent with the pack's "everything is markdown" principle).

Three sections:

**Owned files** — cc-sync manages these (read/write):

```markdown
| Path | Feature | Source template | Content SHA | Written at |
|------|---------|-----------------|-------------|------------|
| .claude/agents/user-auth/backend-engineer.md | user-auth | agents/backend-engineer.md.tmpl | a1b2... | 2026-04-19T14:30:00Z |
```

**Archived** — files whose source feature was removed upstream. cc-sync moves the file to `.claude/archive/<timestamp>/<feature>/` (preserves history) and marks the manifest row:

```markdown
| Path (archive) | Original path | Feature | Archived at |
|---|---|---|---|
| .claude/archive/2026-04-19T14:30:00Z/user-auth/backend-engineer.md | .claude/agents/user-auth/backend-engineer.md | user-auth | 2026-04-19T14:30:00Z |
```

**User-modified** — files where dev-edited SHA differs from manifest SHA. cc-sync logs the divergence; subsequent runs preserve until user explicitly accepts overwrite.

```markdown
| Path | Detected at | Manifest SHA | Current SHA |
|---|---|---|---|
| .claude/agents/user-auth/backend-engineer.md | 2026-04-19T15:00:00Z | a1b2... | 9f8e... |
```

**Lifecycle invariants:**

1. Files not in the manifest are never touched by cc-sync. Manifest is the sole ownership ledger.
2. Manifest is saved incrementally (after each file write) to survive partial-failure mid-sync.
3. Archive is soft — files moved to `.claude/archive/`, not deleted. Consumer can `rm -rf .claude/archive/` when comfortable.
4. User-modified files stay in the Owned section too, but flagged; checksum-prompt fires on next sync until user decides.

---

## Decision 7 — Starter template content (v1.0.0)

Five agent templates + four partials ship at `templates/`:

```
templates/
  VERSION                                  # "1.0.0"
  README.md                                # editing/versioning/fork guide
  agents/
    backend-engineer.md.tmpl               # reads BACKEND_STACK + BACKEND_DATA + BACKEND_API + ARCHITECTURE
    frontend-engineer.md.tmpl              # reads FRONTEND_STACK + TOKENS + COMPONENT_SPECS + FRONTEND_ROUTES
    qa-engineer.md.tmpl                    # reads QA_STRATEGY + TEST_CASES
    product-architect.md.tmpl              # reads PRD + ARCHITECTURE + WORKSTREAMS
    design-reviewer.md.tmpl                # reads TOKENS + COMPONENT_SPECS (a11y-focused, read-only allowed-tools)
  partials/
    testing-rigor-mvp.md.tmpl
    testing-rigor-full.md.tmpl
    shadcn-theme-notes.md.tmpl
    custom-system-notes.md.tmpl
```

**Each agent template body covers:**

- **Frontmatter** — `name: <feature>-<discipline>-engineer`, `description`, `allowed-tools` (varies per agent — backend can Read/Edit/Bash; frontend adds Playwright MCP if present; design-reviewer is Read-only; qa-engineer can Read/Edit/Bash + Playwright)
- **Who you are** — role + feature + what you own
- **Stack** — rendered from STACK.md decision table
- **Conventions** — rules lifted from STACK.md Conventions section (imperative list)
- **Key artifacts** — absolute paths to spec files the agent references
- **Do not** — negative guardrails (e.g., "never hardcode hex colors — use `--primary` from TOKENS.md"; "never create an endpoint not in ARCHITECTURE.md without updating the architecture first")
- **Handoff boundaries** — when to defer to another agent (e.g., qa-engineer hands off visual critique to design-reviewer)

---

## Decision 8 — Integration with existing skills

| Skill | Change | Rationale |
|---|---|---|
| `pm-handoff` | Step 2 menu gains a 6th option: "Bootstrap Claude Code guardrails via `cc-flow`". Step 4 adds a parallel invoke block. Step 5 coordination note mentions cc-flow's role. | Natural entry point after artifacts are produced. |
| `kanban-pickup` | Final step: if `.claude/agents/<feature>/` exists, suggest `@<feature>-<discipline>-engineer` to dev based on which task was picked. | Closes the dev loop — spec → Jira → pickup → agent invocation. |
| `README.md` | Skill count 43 → 45; new "Claude Code guardrails" family in the top diagram; `.cc-templates/` + `.claude/.cc-manifest.md` + `.pm/cc-config.md` added to Artifacts section; new "Guardrails as output" principle; link to `docs/guardrails.md`. | Surfacing the family + its disk footprint. |
| `docs/skills-flow.md` | New mermaid for cc-flow; top-level orchestration gains `pm-handoff → cc-flow` edge; artifacts table gains CC row. | Keeping the flow map complete. |
| `.claude-plugin/plugin.json` + `marketplace.json` | Bump version 1.1.0 → 1.2.0; description adds "+ Claude Code guardrails"; `keywords` adds `claude-code-guardrails`. | Plugin metadata mirrors the scope expansion. |
| `CHANGELOG.md` | New `[1.2.0]` entry under Added. | Convention. |

**Not changed:**

- Any of the 6 content-producing skills (pm-prd, pm-architecture, backend-stack, design-tokens, design-components, qa-strategy). cc-sync is read-only over them by design.
- `PROJECT_PROFILE.md` schema. Presence of `.cc-templates/` + manual `cc-flow` invocation is sufficient signal; no new field needed. Keeps PROJECT_PROFILE stable — no migration prompt for legacy projects.
- backend-tasks, frontend-tasks, qa-tasks, design-tasks. Task field schema stays frozen (kanban-sync contract); cc-sync reuses it.

---

## Decision 9 — Documentation deliverables

User explicitly asked for thorough documentation. Three places:

1. **`templates/README.md`** — in-repo docs for the starter kit. Audience: consumers who just got a fresh `.cc-templates/`. Covers: structure, editing conventions, VERSION meaning, how partials are selected by cc-sync, how to add new templates or partials, what happens on starter updates.

2. **`docs/guardrails.md`** — top-level explainer. Audience: someone evaluating or adopting project-skills. Covers: what cc-flow is, why it exists (the spec→dev-session gap), end-to-end flow walk-through, per-feature scoping rationale, checksum-prompt policy, manifest anatomy, archive lifecycle, starter update policy, how to fork for a company-wide base set of templates, what's in scope for v1 and what's not.

3. **`README.md` additions** — family description in the flow diagram + short blurb + link to `docs/guardrails.md`.

---

## Decision 10 — Rollout and risk mitigations

**Rollout:**

- Ship as v1.2.0 of the plugin.
- No migration needed for projects already using v1.1.x — `cc-flow` is opt-in via `pm-handoff` or direct invocation.

**Mitigations (built into design):**

- **Clobbering user files**: manifest is additive-only. Files absent from manifest are never touched.
- **Silent user-edit loss**: checksum-prompt policy (Decision 4).
- **Template drift**: `.cc-templates/VERSION` compared to `templates/VERSION` in project-skills; warns on divergence.
- **Partial-write corruption**: manifest saved incrementally, re-runs tolerate partial state.
- **Claude Code frontmatter validation**: first sync does a smoke test (create 1 agent, confirm Claude Code picks it up) before rendering the full set.
- **Feature-folder collision with existing `.claude/` content**: on first-write per feature, check for pre-existing non-manifest content; prompt to merge or suffix.
- **Agent sprawl**: subfolder layout (`.claude/agents/<feature>/`) keeps organized. Optional `.claude/agents/README.md` auto-index.
- **Stale agents after spec change**: `cc-sync --preview` shows "N artifacts changed since last sync" by comparing artifact mtimes to manifest timestamps.

---

## Out of scope (explicit, not forgotten)

- Generation of per-feature skills. Skills Claude Code trigger on description match, not feature name — agents are the right shape.
- Remote template source (separate company template repo). Backlog for v1.x.
- `cc-status` as a separate skill. `cc-sync --preview` covers the use case.
- New field in `PROJECT_PROFILE.md` (`ccGuardrails: on | off`). Presence-based detection is sufficient.
- Git pre-commit hook for drift check. Documented as manual practice.
- Project-level (cross-feature) aggregate agents. Revisit if onboarding use case appears.
- Auto-fetching templates from a URL at sync time. Bundled starter + consumer-editable is v1.

---

## Summary of files changed

| File | Change |
|---|---|
| `skills/cc-flow/SKILL.md` | **New** — orchestrator |
| `skills/cc-sync/SKILL.md` | **New** — idempotent sync + manifest |
| `templates/VERSION` | **New** — semver |
| `templates/README.md` | **New** — starter kit docs |
| `templates/agents/backend-engineer.md.tmpl` | **New** |
| `templates/agents/frontend-engineer.md.tmpl` | **New** |
| `templates/agents/qa-engineer.md.tmpl` | **New** |
| `templates/agents/product-architect.md.tmpl` | **New** |
| `templates/agents/design-reviewer.md.tmpl` | **New** |
| `templates/partials/testing-rigor-mvp.md.tmpl` | **New** |
| `templates/partials/testing-rigor-full.md.tmpl` | **New** |
| `templates/partials/shadcn-theme-notes.md.tmpl` | **New** |
| `templates/partials/custom-system-notes.md.tmpl` | **New** |
| `docs/guardrails.md` | **New** — end-to-end docs |
| `docs/plans/2026-04-19-cc-flow-design.md` | This doc |
| `docs/plans/2026-04-19-cc-flow.md` | Implementation plan (next) |
| `skills/pm-handoff/SKILL.md` | Edit — 6th option + cc invoke block |
| `skills/kanban-pickup/SKILL.md` | Edit — suggest agent after context load |
| `README.md` | Edit — skill count + family + artifacts + principle |
| `docs/skills-flow.md` | Edit — mermaid + top-level edge + artifacts table |
| `.claude-plugin/plugin.json` | Edit — version + description + keywords |
| `.claude-plugin/marketplace.json` | Edit — mirror |
| `CHANGELOG.md` | Edit — entry |

**New runtime artifacts (in consumer projects, not this repo):** `.cc-templates/`, `.claude/agents/<feature>/`, `.claude/.cc-manifest.md`, `.pm/cc-config.md`.

---

## Open questions (deferred)

- **Project-level agent aggregation** — revisit when a first user asks for an onboarding flow across features.
- **Remote template source** — revisit if a company explicitly wants central-update without per-consumer-repo template edits.
- **Multi-company templating** — if agencies want per-client template variants, defer.
- **Agent introspection tool** — `cc-diff <feature>` to show what changed between manifest SHA and current file. Nice-to-have, not in v1.
