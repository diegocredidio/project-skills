---
name: kanban-flow
description: Orchestrates Kanban operations over an existing Jira project via the Atlassian MCP server. Two modes — setup (first run: syncs TASKS.md artifacts into Jira) or cycle (ongoing: status reports and dev pickup). Reads `.pm/jira-config.md` (created on first run with `projectKey` + issue type IDs) and invokes `kanban-sync`, `kanban-status`, or `kanban-pickup` as needed. Use when user says "kanban flow", "sync with Jira", "start managing the board", or after PM/Design/Backend/Frontend task artifacts are ready. Not for Scrum (no sprints, no velocity) or for creating Jira projects from scratch.
---

# Kanban Flow — Orchestrator for Jira Sync + Ops

Coordinates the Kanban skills over an existing Jira project. Assumes the project already exists — you provide the `projectKey`, this flow doesn't create projects.

## Modes

**Setup mode** (first run per feature or per new project):
1. Collect / confirm Jira config
2. Invoke `kanban-sync` to push TASKS.md → Jira

**Cycle mode** (ongoing):
- `kanban-status` — board overview (WIP, blocked, done-this-week)
- `kanban-pickup` — dev picks the next card, context loaded, card transitioned to In Progress

## How to run

### Step 1: Setup mode detection

Check for `.pm/jira-config.md`:

- **Missing** → first-run setup. Ask the user:
  1. `projectKey` (e.g., `PROJ`)
  2. Epic issue type name (often `Epic`) — we'll resolve the ID via MCP
  3. Story issue type name (often `Story`)
  4. Task issue type name (often `Task`)
  5. Any custom fields that are required for creating issues in this project
  
  Validate the project exists via the Atlassian MCP capability "get visible Jira projects" or "get project issue types metadata". If the project is not visible / not found, tell the user to fix access and retry — do NOT create the project.
  
  Save config to `.pm/jira-config.md`.

- **Exists** → load it. Confirm with user: "Using Jira config for project `[projectKey]`. Continue?"

### Step 2: Mode dispatch

Ask: "What do you want to do?"

Options:
- **Sync** — push `.pm/<feature>/TASKS.md` (and the three discipline TASKS files if present) to Jira. Invokes `kanban-sync`.
- **Status** — read the board and produce a report. Invokes `kanban-status`.
- **Pickup** — find the next card for a developer and load its context. Invokes `kanban-pickup`.
- **Setup only** — stop after config is confirmed.

### Step 3: Delegate

Invoke the chosen skill. Do not duplicate its behavior here.

### Step 4: Post-action

After any sub-skill completes, offer the next logical step:
- After `kanban-sync`: "Want a status report now? Or pick up the first card?"
- After `kanban-status`: "Want to pick up a card?"
- After `kanban-pickup`: the user is now working — no follow-up needed.

## The `.pm/jira-config.md` file

First-run writes this. Format:

```markdown
# Jira Config

projectKey: PROJ
epicIssueTypeId: 10000
storyIssueTypeId: 10001
taskIssueTypeId: 10002

labels:
  pm: discipline-pm
  design: discipline-design
  backend: discipline-backend
  frontend: discipline-frontend

customFields:
  # If the project requires custom fields on create, add them here.
  # Skills cannot know project-specific custom fields automatically.
  # Example:
  # customfield_10020: { value: "Medium" }   # priority override
```

The file is a user-editable contract. Skills read it; they do not mutate it after first creation.

## Rules

- NEVER create a Jira project. If the projectKey doesn't exist or isn't visible, tell the user to resolve access and retry.
- NEVER create custom fields or modify project workflow — if the project requires custom fields on create and the config doesn't provide them, `kanban-sync` will fail with a clear error message. User edits the config.
- Skills reference Atlassian MCP capabilities by the action they perform (e.g., "create a Jira issue", "search via JQL"), not by exact tool names like `mcp__claude_ai_Atlassian__createJiraIssue` — Claude resolves through whichever Atlassian MCP is available.
- The config file is authoritative. If field IDs change (rare), the user edits the config; skills do not auto-detect changes.
- Kanban flow, not Scrum. No `sprint`, `velocity`, `retrospective` concepts. Cycle time and WIP are the metrics.
