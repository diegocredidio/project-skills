---
name: backend-review
description: Post-build static analysis of implemented backend code against `.backend/<feature>/BACKEND_STACK.md`, `BACKEND_DATA.md`, `BACKEND_API.md`, and `BACKEND_TASKS.md`. Reads diffs and source files; does NOT run tests or execute code. Produces `.backend/<feature>/BACKEND_REVIEW.md` with Critical / Warning / Info findings, coverage matrix, and recommendations. Use when backend-flow invokes it post-build or when user says "backend review", "audit the backend". Not for pre-build planning, for general code review of unrelated changes, or for running test suites.
---

# Backend Review — Static Audit Against Specs

Checks what was built vs. what was declared. Reads code, does not run it.

## Pre-flight

Confirm:
1. An implemented backend source tree exists (e.g., `src/`, `app/`, `internal/`).
2. All four backend specs exist: STACK, DATA, API, TASKS.

If not, abort and tell the user what's missing.

## How to run

### Step 1: Load specs and source

Read the four backend spec files. Identify source roots from BACKEND_STACK.md §3.

Walk the source tree:
- Routes / handlers
- Services
- Repositories
- Middleware
- Migrations
- Tests
- Config (env handling)

### Step 2: Convention audit

For each area, check vs BACKEND_STACK.md conventions:

- **Error propagation:** do services raise typed errors? Does the central handler map them to the declared envelope? Grep for `res.status(xxx).json(raw)` — any that skip the envelope is a finding.
- **Validation:** every route with `params`, `query`, or `body` input has a validation schema applied at the boundary. Find routes that read `req.body.*` without prior validation — finding.
- **Logging:** every log entry has required fields (requestId, userId, timestamp, level, message). Grep for `console.log`, `print(` — findings unless explicitly allowed.
- **Repository isolation:** services import repositories, not ORM clients directly. Grep for `prisma.` / `session.query(` inside service files — findings.
- **Transactions:** multi-step writes declared as transactions at service level, passed to repos as context.

### Step 3: Security audit

- Every endpoint listed in BACKEND_API.md with `auth: user` has auth middleware attached. Find auth-gated routes without the middleware — Critical finding.
- Every mutating route has validation. Unvalidated input reaching service layer — Critical.
- Rate limit middleware engaged per BACKEND_API.md buckets.
- Secrets (JWT_SECRET, DATABASE_URL, Clerk keys) only read from env, never committed. Grep for plaintext matches of suspicious prefixes (`sk_`, `postgres://`, hardcoded bcrypt-looking strings).
- Redaction rules enforced — responses never include `password_hash`, `refresh_token` in user-facing payloads.

### Step 4: Data audit

- Indexes declared in BACKEND_DATA.md present in migrations. Missing indexes — Warning.
- Foreign key ON DELETE behavior matches DATA.md. Mismatch — Warning.
- N+1 patterns: repositories that loop over a list and fire per-item queries — Warning with file:line reference.
- Nullable columns handled in application code (no unhandled null assumptions).

### Step 5: Test audit

- Coverage claim per BACKEND_TASKS.md "Done when" vs actual test files present.
- Integration tests hit a real DB (docker-compose'd), not mocks, per STACK.md.
- No tests for framework internals or generated code (that's in the convention — if present, Info finding).

### Step 6: Declared-vs-implemented matrix

For each BACKEND_TASK:
- Present in code? (files found via task description)
- Tests declared in "Done when"? (test files present?)
- Status: `implemented+tested` / `implemented, tests missing` / `not implemented` / `implemented, deviates from spec`

### Step 7: Write BACKEND_REVIEW.md

Save to `.backend/<feature>/BACKEND_REVIEW.md`:

```markdown
# Backend Review: [Feature Name]

**Date:** [today]
**Specs:** BACKEND_STACK, BACKEND_DATA, BACKEND_API, BACKEND_TASKS
**Source:** `src/`
**Scope:** static analysis only — tests were not executed

## Findings summary

| Severity | Count |
|----------|-------|
| Critical | 2 |
| Warning | 5 |
| Info | 3 |

## Critical

**REVIEW-001 — GET /v1/projects/:id missing auth middleware**
- **Location:** `src/routes/projects.ts:42`
- **Spec:** BACKEND_API.md says `auth: user`, route has no auth attached
- **Impact:** anonymous users can read any project
- **Recommendation:** wrap the route in the `authRequired` plugin

**REVIEW-002 — Password hash leaks in user response**
- **Location:** `src/services/users.ts:58`
- **Spec:** BACKEND_API.md Response for /me says: no `passwordHash`
- **Recommendation:** extract public fields in a `toPublicUser()` helper

## Warning

**REVIEW-003 — N+1 in project list**
- **Location:** `src/routes/projects.ts:18`
- **Pattern:** loops over `projects` and calls `taskRepo.countForProject(p.id)`
- **Recommendation:** add `tasksCount` via `include` + `_count` in Prisma, or move to a single SQL with GROUP BY

**REVIEW-004 — Missing index `tasks(assignee_id)`**
- **Spec:** BACKEND_DATA.md declared it
- **Location:** `prisma/migrations/20260417140530_init/migration.sql`
- **Recommendation:** new migration adding the index

...

## Info

...

## Coverage matrix

| Task | Implementation | Tests | Status |
|------|----------------|-------|--------|
| BACKEND_TASK_001 | `src/` + docker-compose.yml | manual | implemented |
| BACKEND_TASK_002 | `prisma/schema.prisma` + migrations | — | implemented |
| BACKEND_TASK_005 | `src/routes/auth.ts` + `services/auth.ts` | `tests/integration/auth.test.ts` | implemented+tested |
| BACKEND_TASK_006 | `src/routes/projects.ts` + services/repos | no integration test | implemented, tests missing |
| BACKEND_TASK_013 | not found | — | not implemented |

## Recommendations (ordered by severity)

1. Fix REVIEW-001 before merge — security boundary broken.
2. Fix REVIEW-002 — user-facing payload leaks sensitive data.
3. Backfill integration test for BACKEND_TASK_006.
4. Add index migration for REVIEW-004 before first production write.
5. Refactor N+1 in REVIEW-003 before list scales past a few hundred items.
```

### Step 8: Summarize to user

Give a concise summary:

> "Backend review done. [N Critical] / [M Warning] / [K Info]. Biggest issue: [REVIEW-001]. Full report in BACKEND_REVIEW.md."

## Rules

- Never execute tests or run code. Read source only.
- Every finding must cite a file + line (or file + function name if line-number is unreliable).
- Match findings to a declared spec. "I prefer X" is not a finding — "Spec says X, code does Y" is.
- Prioritization: Critical = breaks a security/correctness boundary. Warning = degrades quality or will bite later. Info = style or opportunity.
- Do not fix anything. Review only.
- If a spec ambiguity lets two valid implementations exist, flag it as Info against the spec, not against the code.
