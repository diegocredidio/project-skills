---
name: kanban-pickup
description: For a developer starting a work session — queries Jira via the Atlassian MCP for the next card assigned to the current user (or a user argument), ranks candidates by priority + age + unblocked dependencies, lets the user pick one, transitions it to In Progress with a timestamped comment, then loads the corresponding TASK from local TASKS.md + all related specs into the conversation so the dev can start coding. Use when kanban-flow invokes it or when user says "pickup", "what's next", "start my next card". Not for sprint planning (Kanban), not for bulk assignment.
---

# Kanban Pickup — Next-Card Context Loader

Gets a dev unblocked and coding as fast as possible. Reads Jira, picks a card, loads the matching local spec + dependencies into context.

## Pre-flight

Confirm:
1. `.pm/jira-config.md` exists.
2. `.pm/<feature>/JIRA_MAP.md` exists for the current feature (otherwise we can't link Jira keys back to TASKS.md items).
3. Atlassian MCP reachable.

## How to run

### Step 1: Identify the user

Default: use Atlassian MCP's "get current user info" capability (`atlassianUserInfo` or equivalent) to detect the caller.

If the user passes an explicit assignee argument (e.g., `kanban-pickup @ana`), use that and confirm before proceeding.

### Step 2: Query candidates

JQL (via "search via JQL" MCP capability):

```
project = [PROJ]
AND assignee = [userAccountId]
AND status in ("To Do", "Backlog")
ORDER BY priority DESC, created ASC
```

For each candidate, check: is it unblocked? A card is unblocked when all its `is blocked by` linked issues are `Done`.

If MCP exposes the link list per issue, filter in place. Otherwise fetch links per candidate lazily.

### Step 3: Rank

Order the unblocked candidates:
1. Priority `Highest` / `Blocker` first
2. Then cards whose all blockers just moved to Done (freshly unblocked — ride momentum)
3. Then by age ascending (oldest To Do first — clear inventory)

Cap at top 5.

### Step 4: Present

Show the user:

```
Top candidates for @ana (unblocked, To Do):

1. PROJ-220 "Scaffold project" — priority Medium — 2 days old
   Maps to: BACKEND_TASK_001
   Depends on: none
2. PROJ-245 "Loading boundaries" — priority Low — 5 days old
   Maps to: FRONTEND_TASK_012
   Depends on: FRONTEND_TASK_005, 006, 007 (all Done)
3. PROJ-290 "Empty state component" — priority Medium — 1 day old
   Maps to: DESIGN_TASK_008
   Depends on: DESIGN_TASK_005 (Done)

Pick one (1-3), or "show more", or "none".
```

### Step 5: Confirm and transition

When the user picks (e.g., "1"):

1. Look up local TASK ID from JIRA_MAP.md (`PROJ-220` → `BACKEND_TASK_001`).
2. Confirm: "Transition PROJ-220 to In Progress and load BACKEND_TASK_001 context?"
3. On yes:
   - Via MCP "transition Jira issue" capability: move the card to the "In Progress" transition (use `getTransitionsForJiraIssue` first to resolve the transition id if needed)
   - Via MCP "add comment" capability: post comment `"Picked up via Claude at 2026-04-17T14:30:00Z by @<user>"` — keeps audit trail

### Step 6: Load context

Read and summarize into the conversation:

**The task itself** (from `.<discipline>/<feature>/[DISCIPLINE]_TASKS.md`):
- Title
- Description
- Done when (acceptance criteria)
- Depends on (local + Jira — all should be done)
- Size

**Relevant specs** (load the sections relevant to this task — not the whole file):

- For a `DESIGN_TASK` → relevant sections of TOKENS.md, IA.md, COMPONENT_SPECS.md
- For a `BACKEND_TASK` → relevant sections of BACKEND_STACK.md, BACKEND_DATA.md, BACKEND_API.md
- For a `FRONTEND_TASK` → relevant sections of FRONTEND_STACK.md, FRONTEND_ROUTES.md, COMPONENT_PLAN.md + cross-discipline deps (design tokens + backend API endpoints it wires to)

**Files to touch** (inferred from the task description):
- List the files the task description mentions
- Verify they exist; flag the ones to create

### Step 7: Summary to the user

```
## Now in progress: PROJ-220 — "Scaffold project"

Maps to: BACKEND_TASK_001 (Phase 1 — Foundation)

### Done when
- `npm run dev` serves GET /health
- `npm test` runs Vitest with zero tests (config valid)
- docker-compose up starts Postgres with declared user/db

### Depends on
(none)

### Spec context loaded

From BACKEND_STACK.md:
- Runtime: Node 20 LTS
- Framework: Fastify 4.x
- Folder structure: [tree]
- Env vars: DATABASE_URL, JWT_SECRET, CLERK_SECRET_KEY, LOG_LEVEL

From BACKEND_DATA.md:
- First migration will include: users, projects, tasks
  (not in this task — wait for BACKEND_TASK_002)

### Files to create
- `package.json`
- `tsconfig.json`
- `src/index.ts` (server bootstrap)
- `src/routes/health.ts`
- `docker-compose.yml`
- `.env.example`

### Next step
Say "start" and I'll begin scaffolding. Or ask any question about the spec.
```

### Step 8: Hand off to the user's own workflow

After context is loaded, this skill is done. The user drives implementation from here (possibly invoking other skills or writing code directly).

## Rules

- Never pick a card for the user without their confirmation. Present and wait.
- Never skip the unblocked check. Assigning a blocked card wastes the dev's session.
- Always transition + comment together. One without the other creates an audit gap.
- Context loading is lazy — don't dump the entire spec. Extract only the sections relevant to the chosen task.
- If a picked card has no matching entry in JIRA_MAP.md (created manually in Jira, bypassing sync), flag it: "This card isn't in the map — it may lack local spec context. Continue anyway?"
- If no unblocked candidates exist, say so and suggest:
  - "All your cards are blocked. Top blocker: [list]. Consider `kanban-status` for a broader view."
- Read-then-write: always query before transitioning. Don't assume the user's previous state.
