---
name: kanban-status
description: Reads `.pm/jira-config.md` and queries the Jira board via JQL through the Atlassian MCP. Produces a Kanban flow report — WIP per column, blocked tickets, cards > N days old, cycle time of recently-closed items, done-this-week. Saves `.pm/<feature>/STATUS_<YYYY-MM-DD>.md` and outputs a concise console summary. Use when kanban-flow invokes it or when user says "status report", "what's on the board", "pre-meeting update". Not for Scrum metrics (velocity, burndown) — use cycle time.
---

# Kanban Status — Board Report

Reads Jira and produces a markdown status report suitable for a weekly check-in or stakeholder update.

## Pre-flight

Confirm:
1. `.pm/jira-config.md` exists.
2. Atlassian MCP reachable.

## How to run

### Step 1: Load config

Read `projectKey` from `.pm/jira-config.md`. Optionally accept a `featureFilter` arg — if provided, scope queries to issues in the Epic whose key matches `.pm/<feature>/JIRA_MAP.md`.

### Step 2: Query the board

Via Atlassian MCP's "search via JQL" capability, run these queries. If `featureFilter` is set, append `AND "Epic Link" = <EPIC_KEY>` to each.

**WIP (all active):**
`project = [PROJ] AND statusCategory != Done ORDER BY updated DESC`

**Blocked:**
`project = [PROJ] AND statusCategory != Done AND issueLink in linkedIssuesOf("blocks", ...)` — fall back to a simpler: `project = [PROJ] AND status = "Blocked"` if custom status exists, otherwise filter locally after fetch.

**Stale (> 14 days in In Progress):**
`project = [PROJ] AND status = "In Progress" AND updated < -14d`

**Done this week:**
`project = [PROJ] AND status = Done AND resolutionDate >= -7d ORDER BY resolutionDate DESC`

**Flow count per status:**
Group the WIP query by `status` locally (or use a JQL `count` variant if MCP exposes one).

### Step 3: Compute cycle time (approximate)

For each issue returned by "Done this week":
- cycle time ≈ `resolutionDate - created` (if no transition history available)
- If MCP gives per-status timestamps, prefer: `movedToDone - movedToInProgress`

Compute median + p90 across the set.

### Step 4: Identify flags

- **WIP per assignee > 3** — flag the person + card list
- **Cards without an assignee (In Progress)** — surface them
- **Cards > 14 days in In Progress** — list
- **Blocker without a comment in > 3 days** — if MCP exposes comment timestamps, flag; otherwise skip

### Step 5: Render report

Save to `.pm/<feature>/STATUS_<YYYY-MM-DD>.md` (or `.pm/STATUS_<YYYY-MM-DD>.md` if no feature filter):

```markdown
# Kanban Status — [Project PROJ]

**Date:** 2026-04-17
**Scope:** [all / Epic PROJ-100 — "User Authentication"]
**Source:** Jira via Atlassian MCP

## Flow summary

| Status | Count |
|--------|-------|
| Backlog | 18 |
| To Do | 6 |
| In Progress | 9 |
| Review | 3 |
| Done (total) | 42 |
| Done (this week) | 8 |

## Cycle time (closed this week)

- Median: 3.2 days
- p90: 7.1 days
- Max: 11 days (PROJ-245 — "Apply design tokens")

## Work in progress

### By assignee

| Assignee | Count | Over limit? |
|----------|-------|-------------|
| @diego | 2 | ok |
| @ana | 4 | **over** (limit 3) |
| @pedro | 1 | ok |
| unassigned | 2 | **needs owner** |

### Detail

#### @ana (4 in progress — over limit)
- PROJ-210 "Apply design tokens" — 2 days
- PROJ-220 "Scaffold project" — 3 days
- PROJ-225 "Auth middleware" — 5 days
- PROJ-231 "Register endpoint" — 8 days ⚠️ stale

#### unassigned
- PROJ-245 "Loading boundaries" — needs owner
- PROJ-248 "Breakpoint sweep" — needs owner

## Blockers

| Ticket | Title | Blocked by | Age |
|--------|-------|------------|-----|
| PROJ-231 | Register endpoint | waiting on Clerk sandbox access | 5 days |
| PROJ-290 | Frontend dashboard | blocked by PROJ-220 (in progress) | 3 days |

## Stale (> 14 days In Progress)

| Ticket | Title | Assignee | Days idle |
|--------|-------|----------|-----------|
| PROJ-180 | Legacy data migration | @pedro | 21 |

## Done this week (8)

| Ticket | Title | Assignee | Closed |
|--------|-------|----------|--------|
| PROJ-220 | Scaffold project | @ana | 2026-04-15 |
| ... | | | |

## Recommended actions (machine-inferred)

1. Rebalance: @ana is over WIP limit. Consider pausing PROJ-231 and picking up on PROJ-290 once PROJ-220 clears.
2. Assign PROJ-245 and PROJ-248 — next in pickup queue.
3. Unblock PROJ-231: procure Clerk sandbox today.
4. PROJ-180 (stale 21 days) — check in with @pedro; decide to resume, break down, or cancel.
```

### Step 6: Console summary

Print a concise summary to the user:

> "Board status for PROJ: 9 in progress, 3 in review, 8 done this week. Median cycle time 3.2d. ⚠️ @ana over WIP (4/3), 2 unassigned cards, 1 stale 21d. Full report at .pm/STATUS_2026-04-17.md."

## Rules

- Read only. Never transition, edit, or create issues from this skill — that's `kanban-sync` and `kanban-pickup`.
- Use cycle time, not velocity. Kanban, not Scrum.
- WIP limit default is 3 per assignee — configurable only if the user asks.
- Stale threshold default 14 days. If the team works fast, the default over-flags; if they work slow, it under-flags. Document the default; user can change.
- Timestamps from JQL / MCP may be in UTC — keep UTC throughout the report for clarity.
