---
name: kanban-sync
description: Reads `.pm/<feature>/PRD.md`, `.pm/<feature>/TASKS.md`, and (if present) `.design/<feature>/DESIGN_TASKS.md`, `.backend/<feature>/BACKEND_TASKS.md`, `.frontend/<feature>/FRONTEND_TASKS.md` along with `.pm/jira-config.md`, and creates / updates Jira issues. Hierarchy adapts to project style — team-managed (simplified=true) uses Epic → Task directly; company-managed uses Epic per feature → Story per phase → Task. Writes dependency links from `requires:` fields. Idempotent — re-runs use `.pm/<feature>/JIRA_MAP.md` to update existing instead of duplicating. Use when kanban-flow invokes it or when user says "sync to Jira", "push tasks to Jira", "create issues". Not for creating Jira projects or for running Scrum ceremonies.
---

# Kanban Sync — Tasks to Jira via MCP

Idempotent push of feature artifacts into an existing Jira project. Saves `JIRA_MAP.md` incrementally so any mid-run failure is recoverable.

## Pre-flight

Confirm:
1. `.pm/jira-config.md` exists and has `projectKey`, `projectStyle`, `epicIssueTypeId`, `taskIssueTypeId`, `blocksLinkTypeId`, `effortFieldId`, `effortOptions`.
2. At least `.pm/<feature>/TASKS.md` exists. (Discipline TASKS files are optional — if missing, only PM tasks sync.)
3. The Atlassian MCP server is reachable.

If any fail, abort with a clear message and point the user at `kanban-flow` (setup mode) to fix config.

## Hierarchy by project style

Branch based on `projectStyle` from `.pm/jira-config.md`:

### Team-managed (simplified=true / next-gen)

Team-managed projects **do not have a `Story` issue type available as a parent**. The mapping is:

- **Epic per Phase** (NOT per feature). Each `## Phase N: [Name]` section in TASKS.md becomes one Epic.
- **Task** direct child of Epic. No Story layer.
- **Sub-tasks** optional — only when a single task genuinely combines `[BE]+[FE]+[Design]` concerns; otherwise keep flat.

One feature with 4 phases → 4 Epics in Jira. The feature is not a Jira issue — the project key itself identifies it (one project per major feature is the common pattern for team-managed).

### Company-managed (simplified=false / classic)

Full hierarchy:

- **Epic per feature** (from PRD).
- **Story per discipline+phase** — e.g., "Backend: Foundation".
- **Task per item** — parent Story.

## How to run

### Step 1: Load config + map

- Read `.pm/jira-config.md` → extract everything captured in pre-flight.
- Read `.pm/<feature>/JIRA_MAP.md` if it exists (idempotency map). If missing, this is a first sync.

### Step 2: Load source artifacts

Read in this order:
- `.pm/<feature>/PRD.md` → Epic title + description
- `.pm/<feature>/TASKS.md` → PM stories / tasks
- `.design/<feature>/DESIGN_TASKS.md` if exists → design stories / tasks
- `.backend/<feature>/BACKEND_TASKS.md` if exists → backend stories / tasks
- `.frontend/<feature>/FRONTEND_TASKS.md` if exists → frontend stories / tasks

### Step 3: Dry run preview

Before any MCP write, produce a preview. Preview shape depends on `projectStyle`.

**Team-managed preview:**

```
Will create / update in Jira project [PROJ] (team-managed):

Epics (one per Phase):
  - [CREATE] "Phase 1 — Foundation"          → [N] Tasks
  - [CREATE] "Phase 2 — Core slices"         → [N] Tasks
  - [CREATE] "Phase 3 — Polish"              → [N] Tasks

Tasks (direct children of Epic):
  - [CREATE] "Scaffold project"              (BACKEND_TASK_001)    Effort: Medium
  - [CREATE] "Apply design tokens"           (DESIGN_TASK_001)     Effort: Medium
  - [CREATE] "Error handler middleware"      (BACKEND_TASK_003)    Effort: Small
  - ...

Issue links (Blocks):
  - BACKEND_TASK_005 is blocked by BACKEND_TASK_002, 003, 004
  - FRONTEND_TASK_005 is blocked by FRONTEND_TASK_003, 004, DESIGN_TASK_004, BACKEND_TASK_006
  - ...

Total: [P] Epics + [M] Tasks + [K] links.
```

**Company-managed preview:**

```
Will create / update in Jira project [PROJ] (company-managed):

Epic:
  - [CREATE] "User Authentication" — from PRD.md

Stories (parent = Epic above):
  - [CREATE] "PM: Foundation"       → 3 tasks
  - [CREATE] "Design: Foundation"   → 3 tasks
  - [CREATE] "Backend: Foundation"  → 4 tasks
  - ...

Tasks (parent = Story):
  - [CREATE] "Apply design tokens" (DESIGN_TASK_001) Effort: Medium
  - ...

Total: 1 Epic + [N] Stories + [M] Tasks + [K] links.
```

Ask: "Proceed with the sync?"

### Step 4: Execute in phased batches (NOT all-at-once)

The Atlassian MCP proxy occasionally returns `Invalid content from server` on parallel calls. Batch writes to reduce incidence; sequential retry on individual failures.

**Phase A — Smoke test.** Create 1 Epic + 1 Task under it. Show the user the result (summary + Jira URL) and ask: "Formatting looks right? Proceed?"

**Phase B — All Epics of Phase 1 + their Tasks.** Checkpoint with user.

**Phase C — Remaining Epics (Phase 2, 3, ...) + their Tasks.** One batch.

**Phase D — All `is blocked by` links.** Separate batch; create after all tasks exist.

After each phase, save `.pm/<feature>/JIRA_MAP.md` immediately with what was created. See "Incremental map saves" below.

### Step 5: Issue creation details

#### Epic

- `project`: `projectKey` from config
- `issueType`: `epicIssueTypeId`
- `summary`:
  - Team-managed: `"Phase N — <Phase Name>"` (no feature prefix; project key already identifies the feature)
  - Company-managed: feature name (from PRD title), e.g., `"User Authentication"`
- `description`: follow the Markdown rules in **Step 6** below (CRITICAL — ADF parser has bugs)
- `labels`: `[pm]` (feature labels not needed; `kanban` implicit by project)

Capture the returned Jira key (e.g., `PROJ-100`). Save to `JIRA_MAP.md` IMMEDIATELY.

#### Story (company-managed only)

- `parent`: Epic key from above
- `issueType`: Story ID (if present in config — add during pre-flight if company-managed)
- `summary`: `"[Discipline]: [Phase]"` — e.g., `"Backend: Foundation"`
- `description`: short summary of what the phase covers for that discipline
- `labels`: `[<discipline>]` from config (e.g., `[backend]`)

#### Task

- `parent`: Epic (team-managed) or Story (company-managed)
- `issueType`: `taskIssueTypeId`
- `summary`: task title from the TASKS.md (e.g., `"Scaffold project"`) — **NO** `[TASK-001]` prefix in summary; the local task ID lives in JIRA_MAP.md
- `description`: follow Step 6 below
- `labels`: `[<discipline>]` (`pm` / `design` / `backend` / `frontend`) — single label, no prefixes
- **Custom field `Effort`**: set via `fields: { [effortFieldId]: { id: effortOptions[size] } }` where `size` is `Small` / `Medium` / `Large` from the TASKS.md task size (S/M/L → Small/Medium/Large)
- Any required custom fields from `customFieldsRequired` in config

### Step 6: Markdown description rules (ADF parser — CRITICAL)

The Jira MCP converts our Markdown to ADF (Atlassian Document Format). The parser has specific bugs. Violate these and the description renders garbled or loses content.

**Always:**
- Use `- [ ]` for task lists (hyphen-space-bracket-space-bracket). **NEVER** `* [ ]` (asterisk breaks).
- Put a blank line **before** any list.
- Put Workstream and Effort metadata in **separate paragraphs**, with a blank line between them.
- Separate sections with `##` headings + blank line.

**Never:**
- ❌ Mix `* item` bullets and `- [ ] item` task-list items in the same description. The parser flips modes and truncates.
- ❌ Put sub-bullets under an acceptance-criterion line. Move nested detail to Sub-tasks (team-managed) or to a separate section.
- ❌ Create an issue and edit its description in the same turn in parallel tool calls — the parser caches stale state. Create, then edit in a later turn if needed.

**Canonical template:**

```markdown
## Descrição

<texto livre — 1-3 parágrafos>

**Workstream:** <pm | design | backend | frontend>

**Effort:** <Small | Medium | Large>

## Acceptance Criteria

- [ ] item 1
- [ ] item 2
- [ ] item 3

## Maps to

<FR-XXX, FR-YYY>

## Depends on

- TASK-003 (motivo curto)
- DESIGN_TASK_004 (motivo curto)

## Referências

- .backend/<feature>/BACKEND_API.md#auth
- .design/<feature>/TOKENS.md
```

### Step 7: Create `is blocked by` links

Run this phase AFTER all tasks exist. Parse `Depends on:` / `requires:` from each task in TASKS.md.

**Link direction — get this right:**

`Blocks` link type in Jira:
- `inwardIssue` = the issue **that blocks** (source, the blocker)
- `outwardIssue` = the issue **that is blocked** (target, impeded)

Example: "TASK-010 is blocked by TASK-005" means TASK-005 blocks TASK-010.

```
createIssueLink({
  linkTypeId:  blocksLinkTypeId,          // from config
  inwardIssue:  "UDO-9",                  // TASK-005 — the blocker
  outwardIssue: "UDO-14",                 // TASK-010 — the blocked one
})
```

If you flip these, you get inverted dependencies that look fine in the board but break `kanban-pickup` (which filters "unblocked" candidates).

Cross-discipline links follow the same rule:
- FRONTEND_TASK_005 requires DESIGN_TASK_004 → `inwardIssue` = DESIGN_TASK_004's Jira key (blocker), `outwardIssue` = FRONTEND_TASK_005's Jira key (blocked).

### Step 8: Incremental map saves (CRITICAL for recoverability)

`.pm/<feature>/JIRA_MAP.md` is the single source of truth for idempotency. Save rules:

- **Save immediately** after every successful `createIssue` / `updateIssue` / `createIssueLink` — don't batch writes to the file.
- On a mid-run failure, JIRA_MAP already has everything up to the failure. Re-running `kanban-sync` reads the map and picks up from where it stopped (existing entries → edit-or-skip, missing entries → create).
- If the MCP times out without returning a key, query Jira by `summary` + `projectKey` to reconcile before retry — prevents double-create.

Template:

```markdown
# Jira Map: [Feature Name]

**Project:** PROJ
**Project style:** team-managed
**Last sync:** 2026-04-17 14:30 UTC

## Epics

| Local key | Jira key | Title |
|-----------|----------|-------|
| epic:phase-1 | PROJ-100 | Phase 1 — Foundation |
| epic:phase-2 | PROJ-101 | Phase 2 — Core slices |
| epic:phase-3 | PROJ-102 | Phase 3 — Polish |

## Stories (company-managed only)

| Local key | Jira key | Parent Epic |
|-----------|----------|-------------|
| story:pm:foundation | PROJ-110 | PROJ-100 |
| ... | | |

## Tasks

| Local ID | Jira key | Parent | Title | Status (last seen) |
|----------|----------|--------|-------|--------------------|
| BACKEND_TASK_001 | PROJ-120 | PROJ-100 | Scaffold project | To Do |
| DESIGN_TASK_001 | PROJ-121 | PROJ-100 | Apply design tokens | In Progress |
| ... | | | | |

## Links

| Source (blocker) | Target (blocked) | Type |
|------------------|------------------|------|
| PROJ-120 | PROJ-134 | Blocks |
| ... | | |

## Cancelled (removed upstream, transitioned in Jira)

| Local ID | Jira key | Cancelled on |
|----------|----------|--------------|
| BACKEND_TASK_013 | PROJ-145 | 2026-04-17 |
```

### Step 9: Idempotent update logic

When re-running with a JIRA_MAP.md already present:

- Local task ID exists in map → `editJiraIssue` on the mapped key (update summary / description / Effort if changed)
- Local task ID missing from map but present in TASKS.md → `createJiraIssue` + append to map
- Jira key in map but local task ID missing from TASKS.md (task removed upstream) → `transitionJiraIssue` to `Cancelled` (do NOT delete — preserve history) + move row to the Cancelled section
- Links: reconcile by `(source, target, type)` tuple — never create a duplicate link

### Step 10: Summary

Tell the user:

> "Sync complete. [P] Epics, [M] Tasks, [K] Links. [X created / Y updated / Z cancelled]. Map saved to `.pm/<feature>/JIRA_MAP.md`. Board: https://[your-domain].atlassian.net/jira/software/c/projects/[PROJ]/board"

## Rules

- **Idempotent always.** Re-running must not duplicate issues. JIRA_MAP is the source of truth.
- **Never delete Jira issues.** Upstream-removed tasks transition to `Cancelled`, not deleted.
- **Custom fields are config-driven.** If the project schema requires fields not in `customFieldsRequired`, create-calls will 400 — surface the exact error and stop. User adds to `.pm/jira-config.md`.
- **No project creation, no schema creation.** Skills are write-only on issues, read-only on schema.
- **MCP capability abstraction.** Call capabilities by semantic name ("create Jira issue", "create issue link", "edit issue", "transition issue"). Don't hardcode tool names like `mcp__claude_ai_Atlassian__createJiraIssue` — Claude resolves via whichever Atlassian MCP is available.
- **Dry run before write.** Always show the preview and wait for confirmation.
- **Phased execution + incremental map saves.** Resilience against MCP proxy flakiness.
- **Get link direction right.** `inwardIssue` = blocker, `outwardIssue` = blocked. Flipping this silently inverts dependencies and breaks `kanban-pickup`.
- **Labels minimalistas.** Single label per issue = its discipline. No feature/phase/workstream/effort/task prefixes — all duplicate information already in project key, parent Epic, the custom field, or JIRA_MAP.
