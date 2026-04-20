---
name: cc-flow
description: Orchestrates Claude Code guardrails generation from planning artifacts. Two modes — setup (first run per repo: copies starter templates from project-skills into `.cc-templates/` and writes `.pm/cc-config.md`) or cycle (ongoing: invokes cc-sync to render per-feature agents into `.claude/agents/<feature>/`). Reads `.pm/cc-config.md` and delegates to cc-sync. Use when user says "cc flow", "bootstrap Claude Code guardrails", "sync the agents", or when pm-handoff invokes this skill. Not for writing actual implementation code, not for running agents — this skill only compiles planning artifacts into guardrails.
---

# cc-flow — Orchestrator for Claude Code Guardrails

Coordinates guardrails generation from planning artifacts. Reads artifact trees produced by pm/design/backend/frontend/qa flows and compiles them into per-feature Claude Code agents in `.claude/agents/<feature>/`.

## Modes

**Setup mode** (first run per consumer repo):
1. Run pre-flight checks
2. Copy starter templates from project-skills into `.cc-templates/`
3. Write `.pm/cc-config.md`
4. Transition to Cycle mode

**Cycle mode** (ongoing, per feature):
- `Sync` — invoke `cc-sync` to render agents for a feature
- `Preview` — invoke `cc-sync` in dry-run mode to see what would change
- `Setup only` — stop after confirming config

## How to run

### Step 1: Setup mode detection

Check for `.pm/cc-config.md`.

- **Exists** → load it. Confirm: "Using cc-config (templates version: `[templatesVersion]`, agent scope: `[agentScope]`). Continue?" Skip to Step 2.
- **Missing** → run the Pre-flight checks below before anything else.

### Step 1a: Pre-flight checks (first run only — ABORT if any step fails)

Execute in order:

**1. Confirm planning artifacts are available.** Check that at least one `.pm/<feature>/PRD.md` and `.pm/<feature>/PROJECT_PROFILE.md` exist in the consumer repo. If not, abort:

> "No planning artifacts found. Run `pm-flow` first to produce `.pm/<feature>/PRD.md` and `.pm/<feature>/PROJECT_PROFILE.md`, then return here."

**2. Check for existing `.cc-templates/`.** If `.cc-templates/` already exists, ask: "`.cc-templates/` already exists. Skip copy and go to mode selection?" If yes, skip to Step 2. If no, continue.

**3. Locate the starter templates.** The starter ships with project-skills at `templates/` in the plugin directory. Ask the user to confirm the plugin install path if it cannot be resolved automatically (common path: `~/.claude/plugins/cache/project-skills/project-skills/<version>/templates/`).

**4. Copy `templates/` → `.cc-templates/`.** Copy all files from the starter's `templates/` directory:
- `templates/agents/` → `.cc-templates/agents/`
- `templates/partials/` → `.cc-templates/partials/`
- `templates/VERSION` → `.cc-templates/VERSION`
- `templates/README.md` → `.cc-templates/README.md`

**5. Write `.pm/cc-config.md`** with:
- `templatesVersion` — read from `.cc-templates/VERSION`
- `agentScope` — `feature`
- `overwritePolicy` — `checksum-prompt`
- `createdAt` — current ISO timestamp

See format below.

**6. Show the user a summary** of what was copied:

> "Setup complete. Copied [N] agent templates + [M] partials to `.cc-templates/`. Config written to `.pm/cc-config.md`."

### Step 2: Mode dispatch

Ask: "What do you want to do?"

Options:
- **Sync** — render agents for a feature now. Invokes `cc-sync`.
- **Preview** — dry-run preview showing what would be created/updated. Invokes `cc-sync` in preview mode.
- **Setup only** — stop here. Config is confirmed, templates are in place.

### Step 3: Delegate to cc-sync

Invoke `cc-sync`. Do not duplicate its behavior here. Pass the selected feature slug.

### Step 4: Post-action guidance

After `cc-sync` completes:

> "Agents written to `.claude/agents/<feature>/`. To use them during implementation, run `kanban-pickup` to pick a task — it will suggest the matching agent automatically. Or invoke directly: `@<feature>-backend-engineer`, `@<feature>-frontend-engineer`, etc."

If the user wants to sync another feature, return to Step 2.

## The `.pm/cc-config.md` file

Written on first run after all pre-flight checks pass. Format:

```markdown
# CC Config

templatesVersion: 1.0.0
agentScope: feature
overwritePolicy: checksum-prompt
createdAt: 2026-04-19T14:00:00Z
```

The file is a user-editable contract. `cc-flow` reads it on every run; `cc-sync` reads `overwritePolicy` to determine how to handle user-edited agent files. Skills do not mutate it after first creation.

## Rules

- Never write actual implementation code. This skill only produces guardrail files (`.md` agents).
- Never modify artifact files. The artifact trees `.pm/`, `.design/`, `.backend/`, `.frontend/`, `.qa/` are read-only from cc-flow's perspective.
- First run ALWAYS goes through setup (pre-flight + copy + config). Even if templates already exist, confirm with the user before skipping.
- `.cc-templates/` is consumer-editable. cc-flow copies on first run but never touches it again. All subsequent reads are by `cc-sync`.
- Abort cleanly if `.pm/<feature>/PROJECT_PROFILE.md` is missing for the feature being synced — delegate recovery guidance to `pm-grill` or `pm-handoff`.
- Version drift warning: if `.cc-templates/VERSION` differs from the project-skills starter `templates/VERSION`, warn the user and offer to show a diff. Do NOT auto-overwrite `.cc-templates/`.
