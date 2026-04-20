---
name: cc-sync
description: Renders planning artifacts into per-feature Claude Code agents at `.claude/agents/<feature>/`, using templates from `.cc-templates/` as source. Reads PROJECT_PROFILE.md, PRD, ARCHITECTURE, STACK, TOKENS, COMPONENT_SPECS, API, ROUTES, QA_STRATEGY, TEST_CASES (whichever exist per feature). Idempotent — maintains `.claude/.cc-manifest.md` with SHA-256 of every owned file, uses checksum-prompt policy when user-edited files diverge, archives (not deletes) agents whose source feature was removed upstream. Use when cc-flow invokes it or when user says "sync the guardrails", "update agents for <feature>". Not for editing artifacts, not for executing tests, not for writing implementation code.
---

# cc-sync — Idempotent Guardrails Sync

Renders planning artifacts into per-feature Claude Code agents inside `.claude/agents/<feature>/`. Maintains a manifest at `.claude/.cc-manifest.md` to track ownership, checksums, and archive state.

## Pre-flight checks (ABORT if any step fails)

Execute in order:

1. **`.pm/cc-config.md` must exist.** If missing, abort: "Run `cc-flow` setup first — `.pm/cc-config.md` is required before sync."
2. **`.cc-templates/agents/` must exist.** If missing, abort: "`.cc-templates/` not found. Run `cc-flow` setup to copy starter templates."
3. **Feature slug must be resolved.** If cc-sync was invoked by cc-flow, the slug is passed. If invoked directly, ask: "Which feature do you want to sync agents for? (Use the kebab-case folder name from `.pm/<feature>/`)."
4. **`.pm/<feature>/PROJECT_PROFILE.md` must exist for the selected feature.** If missing, abort: "`.pm/<feature>/PROJECT_PROFILE.md` not found. Run `pm-handoff` or `pm-grill` for this feature first."

## How to run

### Step 1: Load inputs

Read the following files (all optional except PROJECT_PROFILE.md):

**Required:**
- `.pm/<feature>/PROJECT_PROFILE.md` — designMode, uiFramework, testingRigor

**If exists:**
- `.pm/<feature>/PRD.md` — feature description
- `.pm/<feature>/ARCHITECTURE.md` — architecture decisions

**Design artifacts:**
- `.design/<feature>/TOKENS.md`
- `.design/<feature>/COMPONENT_SPECS.md`
- `.design/<feature>/IA.md`

**Backend artifacts:**
- `.backend/<feature>/BACKEND_STACK.md`
- `.backend/<feature>/BACKEND_DATA.md`
- `.backend/<feature>/BACKEND_API.md`

**Frontend artifacts:**
- `.frontend/<feature>/FRONTEND_STACK.md`
- `.frontend/<feature>/FRONTEND_ROUTES.md`
- `.frontend/<feature>/COMPONENT_PLAN.md`

**QA artifacts:**
- `.qa/<feature>/QA_STRATEGY.md`
- `.qa/<feature>/TEST_CASES.md`

**Manifest (for diff):**
- `.claude/.cc-manifest.md` (if exists — first sync if missing)

### Step 2: Extract template variables

Map artifact content to template variables. All variables follow `{{var}}` syntax. Missing artifacts leave their variables as empty strings in the rendered output — agents degrade gracefully.

| Variable | Source artifact | Field |
|---|---|---|
| `{{feature}}` | Derived from the feature slug | — |
| `{{designMode}}` | PROJECT_PROFILE.md | `designMode:` field |
| `{{uiFramework}}` | PROJECT_PROFILE.md | `uiFramework:` field |
| `{{testingRigor}}` | PROJECT_PROFILE.md | `testingRigor:` field |
| `{{stack.runtime}}` | BACKEND_STACK.md | Decision table row "Runtime" |
| `{{stack.framework}}` | BACKEND_STACK.md or FRONTEND_STACK.md | "Framework" row |
| `{{stack.db}}` | BACKEND_STACK.md | "Database" row |
| `{{stack.orm}}` | BACKEND_STACK.md | "ORM" row |
| `{{stack.auth}}` | BACKEND_STACK.md | "Auth" row |
| `{{stack.testing}}` | BACKEND_STACK.md or QA_STRATEGY.md | "Testing" row / tool per layer |
| `{{stack.styling}}` | FRONTEND_STACK.md | "Styling" row |
| `{{stack.state}}` | FRONTEND_STACK.md | "State" row |
| `{{stack.forms}}` | FRONTEND_STACK.md | "Forms" row |
| `{{stack.componentLib}}` | FRONTEND_STACK.md | "Component library" row |
| `{{stack.authClient}}` | FRONTEND_STACK.md | "Auth client" row |
| `{{conventions.backend}}` | BACKEND_STACK.md | "Conventions" section (bullet list) |
| `{{conventions.frontend}}` | FRONTEND_STACK.md | "Conventions" section |
| `{{tokens.primary}}` | TOKENS.md | `--primary` value |
| `{{tokens.background}}` | TOKENS.md | `--background` value |
| `{{tokens.radius}}` | TOKENS.md | `--radius` value |
| `{{shadcnComponents}}` | COMPONENT_SPECS.md | shadcn component inventory list |
| `{{apiEndpoints}}` | BACKEND_API.md | endpoint list |
| `{{routes}}` | FRONTEND_ROUTES.md | route list |
| `{{coverageTargets}}` | QA_STRATEGY.md | coverage targets section |
| `{{testTools}}` | QA_STRATEGY.md | tool per pyramid layer |
| `{{artifactPaths.prd}}` | Derived | Absolute path to `.pm/<feature>/PRD.md` |
| `{{artifactPaths.architecture}}` | Derived | Absolute path to `.pm/<feature>/ARCHITECTURE.md` |
| `{{artifactPaths.workstreams}}` | Derived | Absolute path to `.pm/<feature>/WORKSTREAMS.md` |
| `{{artifactPaths.backendData}}` | Derived | Absolute path to `.backend/<feature>/BACKEND_DATA.md` |
| `{{artifactPaths.backendApi}}` | Derived | Absolute path to `.backend/<feature>/BACKEND_API.md` |
| `{{artifactPaths.backendTasks}}` | Derived | Absolute path to `.backend/<feature>/BACKEND_TASKS.md` |
| `{{artifactPaths.tokens}}` | Derived | Absolute path to `.design/<feature>/TOKENS.md` |
| `{{artifactPaths.componentSpecs}}` | Derived | Absolute path to `.design/<feature>/COMPONENT_SPECS.md` |
| `{{artifactPaths.componentPlan}}` | Derived | Absolute path to `.frontend/<feature>/COMPONENT_PLAN.md` |
| `{{artifactPaths.frontendRoutes}}` | Derived | Absolute path to `.frontend/<feature>/FRONTEND_ROUTES.md` |
| `{{artifactPaths.frontendTasks}}` | Derived | Absolute path to `.frontend/<feature>/FRONTEND_TASKS.md` |
| `{{artifactPaths.qaStrategy}}` | Derived | Absolute path to `.qa/<feature>/QA_STRATEGY.md` |
| `{{artifactPaths.testCases}}` | Derived | Absolute path to `.qa/<feature>/TEST_CASES.md` |
| `{{artifactPaths.qaTasks}}` | Derived | Absolute path to `.qa/<feature>/QA_TASKS.md` |

### Step 3: Select partials

Partials are chosen by cc-sync based on PROJECT_PROFILE.md values — templates themselves have no conditionals.

**For `frontend-engineer.md.tmpl`:**
- `designMode=shadcn-theme` → include partial `shadcn-theme-notes`
- `designMode=custom-system` → include partial `custom-system-notes`
- If designMode is missing → include no design-mode partial; log a warning.

**For `qa-engineer.md.tmpl`:**
- `testingRigor=mvp` → include partial `testing-rigor-mvp`
- `testingRigor=full` → include partial `testing-rigor-full`
- If testingRigor is missing → default to `testing-rigor-mvp`; log a warning.

Read the chosen partial from `.cc-templates/partials/<name>.md.tmpl`. Substitute `{{var}}` inside the partial using the same variable map before insertion.

### Step 4: Render templates

For each agent template in `.cc-templates/agents/`:
1. Read the `.md.tmpl` file.
2. Substitute all `{{var}}` occurrences with values from Step 2.
3. Resolve all `{{> partial-name}}` includes — read the partial, substitute vars, insert in place of the include tag.
4. Compute SHA-256 of the rendered content (hex string).
5. Target output path: `.claude/agents/<feature>/<template-stem>.md` (strip the `.tmpl` extension).

### Step 5: Dry-run preview

Before writing anything, classify each rendered file against the manifest:

| Classification | Condition | Action |
|---|---|---|
| `[CREATE]` | Not in manifest, file doesn't exist on disk | Will write |
| `[UPDATE]` | In manifest, current file SHA matches manifest SHA | Will overwrite silently |
| `[PROMPT]` | In manifest, current file SHA differs from manifest SHA | Will ask user |
| `[SKIP-DELETED]` | In manifest, file missing on disk | Log, skip |

Show the user a preview table:

```
Preview for feature: <feature>

  [CREATE] .claude/agents/<feature>/backend-engineer.md
  [CREATE] .claude/agents/<feature>/frontend-engineer.md
  [CREATE] .claude/agents/<feature>/qa-engineer.md
  [CREATE] .claude/agents/<feature>/product-architect.md
  [CREATE] .claude/agents/<feature>/design-reviewer.md

Total: 5 to create, 0 to update, 0 to prompt, 0 to skip.

Proceed? (y/n)
```

Wait for confirmation before Step 6.

### Step 6: Execute writes

Create `.claude/agents/<feature>/` if it doesn't exist.

For each file:
- **`[CREATE]` / `[UPDATE]`**: write rendered content, record SHA and timestamp in manifest (Owned section).
- **`[PROMPT]`**: show diff summary and ask: "File `.claude/agents/<feature>/<name>.md` was modified since last sync. Keep your edits (k) or overwrite with re-rendered template (o)?" Act on user choice; if `o`, write and update manifest SHA; if `k`, add to User-modified section of manifest.
- **`[SKIP-DELETED]`**: skip write, keep existing manifest row, log to summary.

**Save manifest incrementally** — after each file write, update `.claude/.cc-manifest.md`. Never batch-save at the end; partial saves survive mid-run failures.

### Step 7: Archive pass

After all writes, scan the manifest for agents whose source feature folder no longer exists in `.pm/`:

For each owned file where `.pm/<feature>/` is missing:
1. Create `.claude/archive/<ISO-timestamp>/<feature>/`.
2. Move the file from `.claude/agents/<feature>/<name>.md` to the archive path.
3. Update the manifest row: move from Owned to Archived section, record `archivedAt`.

Log: "Archived N agents for feature `<feature>` (source removed upstream)."

### Step 8: Report

```
Sync complete for feature: <feature>

  Created:  N
  Updated:  M
  Prompted: K (user-edited — kept/overwritten per choice)
  Archived: X
  Skipped:  Y

Manifest: .claude/.cc-manifest.md
Agents:   .claude/agents/<feature>/

To use: invoke @<feature>-backend-engineer, @<feature>-frontend-engineer, etc.
Run kanban-pickup to pick a task — it will suggest the matching agent automatically.
```

## Template variable reference

See Step 2 above for the full variable table. Summary of sources:

- **Slug / profile**: `{{feature}}`, `{{designMode}}`, `{{uiFramework}}`, `{{testingRigor}}` — from feature slug and PROJECT_PROFILE.md.
- **Backend stack**: `{{stack.runtime}}`, `{{stack.framework}}`, `{{stack.db}}`, `{{stack.orm}}`, `{{stack.auth}}`, `{{stack.testing}}`, `{{conventions.backend}}` — from BACKEND_STACK.md.
- **Frontend stack**: `{{stack.styling}}`, `{{stack.state}}`, `{{stack.forms}}`, `{{stack.componentLib}}`, `{{stack.authClient}}`, `{{conventions.frontend}}` — from FRONTEND_STACK.md.
- **Design tokens**: `{{tokens.primary}}`, `{{tokens.background}}`, `{{tokens.radius}}`, `{{shadcnComponents}}` — from TOKENS.md and COMPONENT_SPECS.md. `{{shadcnComponents}}` is only populated when `designMode=shadcn-theme`; empty string otherwise.
- **QA**: `{{coverageTargets}}`, `{{testTools}}` — from QA_STRATEGY.md.
- **Artifact paths**: all `{{artifactPaths.*}}` — derived as absolute paths.

## The `.claude/.cc-manifest.md` file

Path: `.claude/.cc-manifest.md`. Markdown table format (consistent with the pack's "everything is markdown" principle). Three sections:

### Owned

cc-sync manages these files (read + write):

```markdown
## Owned

| Path | Feature | Source template | Content SHA | Written at |
|------|---------|-----------------|-------------|------------|
| .claude/agents/user-auth/backend-engineer.md | user-auth | agents/backend-engineer.md.tmpl | a1b2c3d4... | 2026-04-19T14:30:00Z |
| .claude/agents/user-auth/frontend-engineer.md | user-auth | agents/frontend-engineer.md.tmpl | e5f6g7h8... | 2026-04-19T14:30:00Z |
```

### Archived

Files whose source feature was removed upstream. Original files moved to `.claude/archive/`:

```markdown
## Archived

| Archive path | Original path | Feature | Archived at |
|---|---|---|---|
| .claude/archive/2026-04-20T10:00:00Z/user-auth/backend-engineer.md | .claude/agents/user-auth/backend-engineer.md | user-auth | 2026-04-20T10:00:00Z |
```

### User-modified

Files where the developer's edits diverged from the last cc-sync SHA. Tracked for visibility; overwrite requires explicit user confirmation:

```markdown
## User-modified

| Path | Detected at | Manifest SHA | Current SHA |
|---|---|---|---|
| .claude/agents/user-auth/backend-engineer.md | 2026-04-19T15:00:00Z | a1b2c3d4... | 9f8e7d6c... |
```

## Rules

- **Manifest is authoritative for ownership.** cc-sync never touches files absent from the manifest. Files not in the Owned section are invisible to cc-sync.
- **Manifest saves incrementally.** After each file write, the manifest is updated. Re-runs tolerate partial-write state from a previous failure.
- **Archive, don't delete.** When a feature is removed upstream, owned agent files move to `.claude/archive/<timestamp>/`. They are never deleted by cc-sync. Consumers can prune the archive directory manually when comfortable.
- **checksum-prompt policy.** User-edited files (SHA diverged from manifest) are never silently overwritten. cc-sync always prompts before overwriting. "Keep edits" leaves the file intact and logs to User-modified section.
- **Read-only over artifact trees.** cc-sync writes only to `.claude/` and its sub-directories. It never modifies `.pm/`, `.design/`, `.backend/`, `.frontend/`, or `.qa/`.
- **First sync smoke test.** On the first sync for a feature (no manifest entry exists), create and verify 1 agent first. Confirm Claude Code recognizes the agent before writing the remaining 4. Abort cleanly if frontmatter validation fails.
- **No `{{#if}}` or `{{#each}}` in templates.** Branching lives in cc-sync (Step 3 partial selection). If a template file contains conditional syntax, log a warning and render the raw tag rather than failing.
- **Template version drift warning.** On every run, compare `.cc-templates/VERSION` to the project-skills starter `templates/VERSION` (if resolvable). If they differ, warn: "Your `.cc-templates/` is at v<X> but the project-skills starter is at v<Y>. Run `cc-flow` to review the diff." Do not auto-update.
