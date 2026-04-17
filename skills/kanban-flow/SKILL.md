---
name: kanban-flow
description: Orchestrates Kanban operations over an existing Jira project via the Atlassian MCP server. Two modes — setup (first run: runs pre-flight checks and invokes kanban-sync) or cycle (ongoing: status reports and dev pickup). Reads `.pm/jira-config.md` (created on first run with `projectKey`, issue type IDs, link type IDs, custom field IDs, and project style team-managed vs company-managed) and invokes `kanban-sync`, `kanban-status`, or `kanban-pickup` as needed. Use when user says "kanban flow", "sync with Jira", "start managing the board", or after PM/Design/Backend/Frontend task artifacts are ready. Not for Scrum (no sprints, no velocity) or for creating Jira projects from scratch.
---

# Kanban Flow — Orchestrator for Jira Sync + Ops

Coordinates the Kanban skills over an existing Jira project. Assumes the project already exists — you provide the `projectKey`, this flow doesn't create projects.

## Modes

**Setup mode** (first run per project):
1. Run pre-flight checks
2. Collect / confirm Jira config
3. Invoke `kanban-sync` to push TASKS.md → Jira

**Cycle mode** (ongoing):
- `kanban-status` — board overview (WIP, blocked, done-this-week)
- `kanban-pickup` — dev picks the next card, context loaded, card transitioned to In Progress

## How to run

### Step 1: Setup mode detection

Check for `.pm/jira-config.md`.

- **Exists** → load it. Confirm: "Using Jira config for project `[projectKey]` (style: `[team-managed / company-managed]`). Continue?" Skip to Step 2.
- **Missing** → run the Pre-flight checks below before anything else.

### Step 1a: Pre-flight checks (first run only — ABORT if any step fails)

Execute these in order. Each uses a semantic MCP capability (the exact tool name varies by MCP server — Claude resolves):

**1. Resolve Atlassian cloud.** Call `getAccessibleAtlassianResources`. Capture `cloudId`. If the call fails or returns empty, abort: "No Atlassian resources accessible. Check MCP authentication and retry."

**2. Confirm authenticated user.** Call `atlassianUserInfo`. Show the user: "Authenticated as `<name>` (`<email>`). Continue?" This prevents accidentally syncing to the wrong workspace.

**3. Ask for `projectKey`.** Validate it exists and is reachable by calling `getJiraProjectIssueTypesMetadata` (or similar "issue types for project" capability). Capture:
- `projectTypeKey` (e.g., `software`)
- `simplified` (boolean) — `true` = team-managed / next-gen, `false` = company-managed / classic
- list of issue types and their IDs

If the call returns "You do not have permission to create issues" or equivalent, orient the user with both paths before aborting:
- **(A)** Re-authenticate the MCP with a Jira account that has `Create Issues` permission on this project.
- **(B)** Ask your Jira admin to add the currently-authenticated account as a project member with role `Create Issues` (or higher, like `Member`).

**4. Verify required issue types.** From the list above, confirm `Epic` exists. Confirm `Tarefa` OR `Task` exists (both map the same — use whichever is present; localized instances can be either). Capture their IDs as `epicIssueTypeId` and `taskIssueTypeId`. If either missing, abort: "Project is missing `Epic` or `Task/Tarefa` issue types. Add them in Project settings → Issue types."

**5. Verify link types.** Call "get issue link types" MCP capability. Confirm `Blocks` link type exists (name in English even on localized instances). Capture its ID as `blocksLinkTypeId`. If missing, abort with instructions to enable it.

**6. Verify or create custom field `Effort`.** Via "get issue type metadata with fields" for `Task/Tarefa`, check if a custom field named `Effort` is present. If yes, capture `effortFieldId` + option IDs (Small/Medium/Large). If no, instruct the user to create it in the UI (skills do NOT modify project schema):

  > To add the Effort field:
  > 1. **Jira Settings → Issues → Custom fields → Add custom field**
  > 2. Type: **Dropdown / Select (single)** — NOT "Short text" (short text allows typos; dropdown constrains values)
  > 3. Name: `Effort`
  > 4. Options: `Small`, `Medium`, `Large`
  > 5. Associate the field with the relevant screens for `Task/Tarefa` issue type on this project
  > 6. Tell me `ok` to re-check — I will capture the field ID automatically.

Loop until the field is found. Capture `effortFieldId` and the option IDs.

**7. Write `.pm/jira-config.md`** with everything captured. See format below.

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

Written on first run after all pre-flight checks pass. Format:

```markdown
# Jira Config

cloudId: abc123-def456
projectKey: PROJ
projectTypeKey: software
projectStyle: team-managed       # team-managed | company-managed — determines sync hierarchy

# Issue types (resolved via MCP during pre-flight)
epicIssueTypeId: 10000
taskIssueTypeId: 10002            # either `Task` or `Tarefa` — same mapping

# Link types
blocksLinkTypeId: 10100           # `Blocks` (inward=blocker / outward=blocked)

# Custom fields
effortFieldId: customfield_10050
effortOptions:
  Small:  10200
  Medium: 10201
  Large:  10202

# Labels applied to every issue (no prefixes — Jira project key already identifies feature)
labels:
  pm: pm
  design: design
  backend: backend
  frontend: frontend

# Any project-specific required fields on Create go here
customFieldsRequired:
  # customfield_10020: { id: "10301" }   # e.g., required "Priority" dropdown
```

The file is a user-editable contract. Skills read it; they do not mutate it after first creation.

## Rules

- NEVER create a Jira project. If the projectKey doesn't exist or isn't visible, tell the user to resolve access and retry.
- NEVER modify project schema (custom fields, workflows, screens). Skills can only detect and reuse; creation is user-driven via the UI.
- Skills reference Atlassian MCP capabilities by the action they perform ("create a Jira issue", "search via JQL"), not by exact tool names like `mcp__claude_ai_Atlassian__createJiraIssue` — Claude resolves through whichever Atlassian MCP is available.
- The config file is authoritative. If field IDs change (rare), the user edits the config; skills do not auto-detect changes outside first-run pre-flight.
- Kanban flow, not Scrum. No `sprint`, `velocity`, `retrospective` concepts. Cycle time and WIP are the metrics.
- Pre-flight is the only phase that can abort cleanly. Once `kanban-sync` starts writing, failures become partial-write states recoverable via `JIRA_MAP.md`.
