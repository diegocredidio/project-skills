---
name: backend-tasks
description: Reads `.backend/<feature>/BACKEND_STACK.md`, `BACKEND_DATA.md`, `BACKEND_API.md`, and `BACKEND_BRIEF.md` and produces `.backend/<feature>/BACKEND_TASKS.md` — dependency-ordered vertical slices. Each task is a thin end-to-end increment (migration + repository + service + route + tests) that one engineer can ship in 1–3 days. Use when backend-flow invokes it or when user says "backend tasks", "break down the backend work". Not for horizontal "build all models" task lists or for cross-discipline task generation.
---

# Backend Tasks — Vertical-Sliced Implementation Plan

Turns the resolved backend plan into a sequenced work list a dev team can execute without re-reading every spec.

## How to run

### Step 1: Load inputs

Read:
- `.backend/<feature>/BACKEND_STACK.md`
- `.backend/<feature>/BACKEND_DATA.md`
- `.backend/<feature>/BACKEND_API.md`
- `.backend/<feature>/BACKEND_BRIEF.md` (tasks-already-defined section)

Abort if any are missing.

### Step 2: Group into phases

1. **Foundation** — scaffold per BACKEND_STACK §3, base migrations, core middleware (auth, error handler, logger), health check
2. **Core slices** — each user-visible backend capability (register, login, create project, list tasks, etc.)
3. **Secondary** — admin endpoints, exports, reports
4. **Observability / hardening** — metrics, tracing, rate limiting, security headers, backup strategy

### Step 3: Write each slice as a vertical task

A slice = one thin end-to-end feature touching migration + repository + service + route + tests. Never split layers into separate tasks.

Per task:
- **Title** (imperative)
- **Description** (what the user can do once shipped)
- **Done when** (acceptance criteria — observable)
- **Maps to** FR-XXX / endpoint / entities
- **Depends on** prior task IDs
- **Effort** (S / M / L)

### Step 4: Order by dependency

Foundation → auth → first protected feature → subsequent features → secondary → observability.

Parallelizable slices get flagged.

### Step 5: Produce dependency graph

Plain text. ASCII arrows or bullet-indented list.

### Step 6: Write BACKEND_TASKS.md

Save to `.backend/<feature>/BACKEND_TASKS.md`:

```markdown
# Backend Tasks: [Feature Name]

## Summary

| Phase | Count | Est. effort |
|-------|-------|-------------|
| Foundation | 4 | ~4 days |
| Core slices | 8 | ~15 days |
| Secondary | 3 | ~5 days |
| Hardening | 4 | ~4 days |

## Phase 1 — Foundation

### BACKEND_TASK_001 — Scaffold project

**Description:** Create the folder structure defined in BACKEND_STACK.md §3. Set up tsconfig, eslint, prettier, vitest config, docker-compose.yml for Postgres.
**Done when:**
- `npm run dev` starts the server and serves GET /health returning `{ok: true}`
- `npm test` runs Vitest with zero tests passing (config valid)
- docker-compose up starts a Postgres with the declared user/db
**Maps to:** infra
**Depends on:** none
**Size:** M

### BACKEND_TASK_002 — Apply initial schema

**Description:** Translate BACKEND_DATA.md entities into Prisma schema; generate and run the first migration.
**Done when:**
- `prisma migrate dev` runs clean from empty DB
- All entities in DATA.md present with correct types and FKs
- Indexes declared in DATA.md are in the migration SQL
**Maps to:** BACKEND_DATA.md
**Depends on:** 001
**Size:** M

### BACKEND_TASK_003 — Error handler + logger middleware

**Description:** Central error envelope per BACKEND_API.md, structured JSON logger with requestId propagation.
**Done when:** unhandled error returns the canonical envelope; log entries include required fields.
**Maps to:** BACKEND_API.md error envelope
**Depends on:** 001
**Size:** S (parallel with 002)

### BACKEND_TASK_004 — Auth middleware

**Description:** Clerk JWT verification → `req.user`, `auth: true` route plugin.
**Done when:** protected route returns 401 without token; returns user payload when valid token attached.
**Depends on:** 001
**Size:** M (parallel with 002, 003)

## Phase 2 — Core slices

### BACKEND_TASK_005 — User register + login

**Description:** POST /v1/auth/login, hashed passwords (or delegate to Clerk), return access + refresh.
**Done when:**
- Integration test: register → login → receive token → call protected route
- Invalid credentials return 401 with proper error envelope
- Rate-limit bucket engaged (100 req/min)
**Maps to:** FR-001, FR-002 / POST /v1/auth/login
**Depends on:** 002, 003, 004
**Size:** L

### BACKEND_TASK_006 — Create + list my projects

**Description:** POST /v1/projects, GET /v1/projects (cursor pagination).
**Done when:**
- Authenticated user creates project with valid input; receives the project in response
- List returns only projects owned by current user
- Pagination: cursor correctly advances; limit respected
- Integration tests pass against real DB
**Maps to:** FR-010, FR-011 / POST /v1/projects, GET /v1/projects
**Depends on:** 005
**Size:** M

### BACKEND_TASK_007 — Project detail + update + delete

... (similar)

### BACKEND_TASK_008 — Tasks CRUD within project

... (similar)

## Phase 3 — Secondary

### BACKEND_TASK_013 — Export project as JSON

...

## Phase 4 — Hardening

### BACKEND_TASK_016 — Rate limiter per bucket

**Description:** Implement per-user and per-IP buckets per BACKEND_API.md Conventions.
**Done when:** load test exceeding bucket returns 429 with canonical error envelope.
**Depends on:** 003
**Size:** M

### BACKEND_TASK_017 — Structured metrics

... (Prometheus endpoint or vendor SDK)

### BACKEND_TASK_018 — Backup + restore runbook

... (document DB backup process, verify restore in staging)

## Dependency graph

```
001 ─┬─ 002 ─ 005 ─ 006 ─ 007 ─ 008 ─ 013
     ├─ 003 ─┤
     └─ 004 ─┘
             └─ 016 ─ 017
                      └─ 018
```

## Parallelizable

- 002 ∥ 003 ∥ 004 (independent middleware/schema work)
- 006 ∥ 007 ∥ 008 (different entity slices, independent)
- 016 ∥ 017 (orthogonal)
```

### Step 7: Hand off

Tell the user: "Backend tasks ordered ([N] total, [P] parallelizable). Hand this to a dev team. Return to `backend-review` once code is landing for static audit against the specs."

## Rules

- A slice must touch every relevant layer (migration + repository + service + route + tests). Never split "build repository" and "build service" into separate tasks for the same feature.
- "Done when" criteria must be observable without reading code — integration test passing, endpoint returning expected shape, etc.
- Foundation always comes first. Shipping a protected endpoint without auth middleware in place is not permitted.
- Every task maps to a concrete FR or endpoint from the specs, or is explicitly flagged as infrastructure.
- Size S means 1 day, M means 2–3 days, L means 4+ days. Tasks > L get broken down until they fit.
