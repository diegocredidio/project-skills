---
name: kanban-sync
description: Reads `.pm/<feature>/PRD.md`, `.pm/<feature>/TASKS.md`, and (if present) `.design/<feature>/DESIGN_TASKS.md`, `.backend/<feature>/BACKEND_TASKS.md`, `.frontend/<feature>/FRONTEND_TASKS.md` along with `.pm/jira-config.md`, and creates / updates Jira issues: one Epic per feature, Stories per discipline phase, Tasks per TASK item. Writes dependency links from `requires:` fields. Idempotent — re-runs update existing issues using `.pm/<feature>/JIRA_MAP.md`. Use when kanban-flow invokes it or when user says "sync to Jira", "push tasks to Jira", "create issues". Not for creating Jira projects or for running Scrum ceremonies.
---

# Kanban Sync — Tasks to Jira via MCP

Idempotent push of feature artifacts into an existing Jira project.

## Pre-flight

Confirm:
1. `.pm/jira-config.md` exists and has `projectKey`.
2. At least `.pm/<feature>/TASKS.md` exists. (Discipline TASKS files are optional — if missing, only PM tasks sync.)
3. The Atlassian MCP server is reachable (tool with capability "get visible Jira projects" or similar is available).

If any fail, abort with a clear message.

## How to run

### Step 1: Load config + map

- Read `.pm/jira-config.md` → extract `projectKey`, issue type IDs, labels, custom fields.
- Read `.pm/<feature>/JIRA_MAP.md` if it exists (idempotency map). If missing, this is a first sync.

### Step 2: Load source artifacts

Read in this order:
- `.pm/<feature>/PRD.md` → Epic title + description
- `.pm/<feature>/TASKS.md` → PM stories / tasks
- `.design/<feature>/DESIGN_TASKS.md` if exists → design stories / tasks
- `.backend/<feature>/BACKEND_TASKS.md` if exists → backend stories / tasks
- `.frontend/<feature>/FRONTEND_TASKS.md` if exists → frontend stories / tasks

### Step 3: Dry run preview

Before any MCP write, produce a preview:

```
Will create / update in Jira project [PROJ]:

Epic:
  - [CREATE] "User Authentication" — from PRD.md

Stories (parent = Epic above):
  - [CREATE] "PM: Foundation" — 3 tasks
  - [CREATE] "Design: Foundation" — 3 tasks
  - [CREATE] "Backend: Foundation" — 4 tasks
  - [CREATE] "Frontend: Foundation" — 4 tasks
  - [CREATE] "Backend: Core slices" — 8 tasks
  - ...

Tasks (parent = Story; examples):
  - [CREATE] "Apply design tokens" (DESIGN_TASK_001)
  - [CREATE] "Scaffold project" (BACKEND_TASK_001)
  - ...

Issue links (blocks / is blocked by):
  - BACKEND_TASK_005 → is blocked by BACKEND_TASK_002, 003, 004
  - FRONTEND_TASK_005 → is blocked by FRONTEND_TASK_003, FRONTEND_TASK_004, DESIGN_TASK_004, BACKEND_TASK_006
  - ...

Total: 1 Epic + [N] Stories + [M] Tasks + [K] links.
```

Ask: "Proceed with the sync?"

### Step 4: Execute (write) via MCP

If confirmed:

**Create / update Epic.** Use Atlassian MCP's "create Jira issue" (or "edit" if already mapped) with:
- project: `projectKey`
- issueType: Epic
- summary: feature name (from PRD title)
- description: 1-2 paragraph summary of the PRD (not the full PRD — Jira descriptions are not designed for long markdown)
- labels: `pm-skills`

Capture the returned issue key (e.g., `PROJ-100`). Record in JIRA_MAP.md.

**Create / update Stories** (one per discipline-phase pairing where tasks exist):
- Parent: Epic key above
- issueType: Story
- summary: `"[Discipline]: [Phase]"` — e.g., "Backend: Foundation"
- description: summary of what the phase covers
- labels: discipline label from config

**Create / update Tasks** (one per TASK item):
- Parent: the Story above
- issueType: Task
- summary: task title from the TASKS.md
- description: Markdown converted — include "Done when" criteria, "Depends on", any spec refs
- labels: discipline label
- Any custom fields from `.pm/jira-config.md`

**Idempotent update logic:**
- If a local task ID (e.g., `BACKEND_TASK_005`) maps to an existing Jira key in JIRA_MAP.md → edit that issue instead of create
- If a local task ID is **missing from TASKS.md** (was deleted upstream) → transition the Jira issue to `Cancelled` (do not delete — preserve history)
- If a local task ID is **new** (not in the map) → create + record

### Step 5: Create issue links for dependencies

Parse `requires:` / `Depends on:` fields in each TASKS.md. For each dependency:
- Look up the source and target Jira keys in JIRA_MAP.md
- Use Atlassian MCP's "create issue link" with `linkType: is blocked by` (or `blocks`, as appropriate)

Cross-discipline links follow the same rule:
- FRONTEND_TASK_005 requires DESIGN_TASK_004 → create link

### Step 6: Update JIRA_MAP.md

Save `.pm/<feature>/JIRA_MAP.md`:

```markdown
# Jira Map: [Feature Name]

**Project:** PROJ
**Epic:** PROJ-100
**Last sync:** 2026-04-17 14:30 UTC

## Stories

| Local key | Jira key | Title |
|-----------|----------|-------|
| story:pm:foundation | PROJ-101 | PM: Foundation |
| story:design:foundation | PROJ-102 | Design: Foundation |
| story:backend:foundation | PROJ-103 | Backend: Foundation |
| ... | | |

## Tasks

| Local ID | Jira key | Title | Status (last seen) |
|----------|----------|-------|--------------------|
| PM_TASK_001 | PROJ-200 | ... | To Do |
| DESIGN_TASK_001 | PROJ-210 | Apply design tokens | In Progress |
| BACKEND_TASK_001 | PROJ-220 | Scaffold project | Done |
| ... | | | |

## Cancelled (removed upstream, preserved in Jira)

| Local ID | Jira key | Cancelled on |
|----------|----------|--------------|
| BACKEND_TASK_013 | PROJ-233 | 2026-04-17 |
```

### Step 7: Summary

Tell the user:

> "Sync complete. Epic: PROJ-100. [N Stories], [M Tasks], [K Links]. [X created / Y updated / Z cancelled]. Map saved to JIRA_MAP.md. Board: https://[your-domain].atlassian.net/jira/software/c/projects/PROJ/board"

## Rules

- **Idempotent always.** Re-running this skill must not duplicate issues. If JIRA_MAP.md is missing but issues with matching titles exist, that's a user-recoverable state — ask before doing anything that could double.
- **Never delete Jira issues.** Tasks removed upstream get transitioned to `Cancelled`, not deleted.
- **Custom field support** is config-driven. If the Jira project requires custom fields on create and the config doesn't provide them, MCP create will 400 — surface the exact error to the user and stop.
- **No project creation.** If the project doesn't exist, abort — `kanban-flow` already validated this, but double-check.
- **MCP capability abstraction.** Call "create Jira issue", "create issue link", "edit issue", "transition issue" by semantic name. Don't hardcode tool names like `mcp__claude_ai_Atlassian__createJiraIssue` in instructions — Claude resolves through whichever Atlassian MCP is available.
- **Dry run before write.** Always show the preview and wait for confirmation. Prevents surprise writes.
