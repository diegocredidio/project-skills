---
name: backend-brief-intake
description: Reads `.backend/<feature>/BACKEND_BRIEF.md` (from pm-handoff or fallback to `.pm/<feature>/ARCHITECTURE.md`), audits the existing codebase, classifies each backend concern as clear/ambiguous/missing, runs an embedded technical grill (gap-fill only) on the ambiguous and missing items, and produces `.backend/<feature>/BACKEND_INTAKE.md`. Use when backend-flow invokes it or when user says "intake the backend brief", "audit the backend". Not for generic backend consulting or for replanning PM-level scope.
---

# Backend Brief Intake — Brief + Codebase Reality + Gap-Fill Grill

Turns an incoming backend brief into an actionable starting point for `backend-stack`.

## How to run

### Step 1: Load the brief

Read `.backend/<feature>/BACKEND_BRIEF.md`. If absent, try `.pm/<feature>/ARCHITECTURE.md`. If neither, abort and ask the user to paste one.

Extract:
- Feature name and purpose
- Stack hints (language/framework/db if declared)
- Data model (entities named)
- API surface (endpoints hinted or declared)
- Performance / scale / security constraints

### Step 2: Audit the codebase

Read:
- **Node.js:** `package.json`, `tsconfig.json`, `src/`, existing route files
- **Python:** `requirements.txt` / `pyproject.toml`, `app/` or `src/` tree, `alembic/` or migration dirs
- **Go:** `go.mod`, `cmd/` and `internal/` trees
- **Ruby:** `Gemfile`, `app/` tree
- Any `Dockerfile`, `docker-compose.yml`, `.env.example`
- Existing migrations under `prisma/migrations/`, `alembic/versions/`, `db/migrate/`, etc.
- Existing auth / middleware / repository patterns

### Step 2.5: Parent artifact read (lineage-only)

If `.pm/<feature>/PARENT.md` exists, this feature is an evolution of a parent slug. Read the parent slug from `PARENT.md`'s H1 (`# Parent: <parent-slug>`), then also read:
- `.backend/<parent-slug>/BACKEND_STACK.md` — runtime, framework, DB, ORM, validation, auth
- `.backend/<parent-slug>/BACKEND_DATA.md` — entities, relationships, migration history
- `.backend/<parent-slug>/BACKEND_API.md` — endpoint inventory, conventions, error format

Treat these as ✅ clear inheritance. In Step 3, classify the inherited concerns as ✅ clear — do NOT re-interrogate runtime choice, framework, DB engine, ORM, validation, auth strategy, or API style when those were resolved in the parent. At emission time (Step 6), annotate each inherited concern with "(inherited from parent)" in the Source column of the "What's clear" table. The embedded gap-fill grill (Step 4) asks only about concerns the child INTRODUCES.

Record parent artifact paths in the output's "Parent baseline" section alongside "Codebase reality" — user and downstream skills see both.

If the parent folder is missing any of the three files, proceed with what's available and note the absence.

If `PARENT.md` does not exist, this step is a no-op — continue to Step 3 in standalone mode.

### Step 3: Classify every backend concern

For each of these, mark ✅ clear / ⚠️ ambiguous / ❌ missing:

- Runtime + version
- Framework
- Database engine
- ORM or query layer
- Validation library
- Auth strategy (provider vs roll-your-own, session vs JWT)
- Serverless vs long-running
- API style (REST / GraphQL / tRPC)
- Versioning strategy
- Rate limiting
- Background jobs / queues
- File storage
- Observability (logging, metrics, tracing)
- Testing approach
- Hosting target

### Step 4: Embedded gap-fill grill

Ask one question at a time, ordered Critical → Important, **only for items marked ⚠️ or ❌**. Never re-ask ✅ items, and never re-grill concerns that Step 2.5 classified as inherited from the parent.

For each question, offer a default the user can accept with "yes":

- "Serverless or long-running? Default for most projects: long-running + container."
- "Session-based or JWT? Default: stateless JWT access + refresh token."
- "REST or GraphQL? Default: REST unless the frontend needs relational queries over the wire."
- "Auth provider or roll-your-own? Default: provider (Clerk / Auth0 / Supabase Auth) unless regulatory constraint."
- "Background jobs — needed? Default: no; add only when a feature demands it."

If the user answers vaguely, probe with "concretely, which?" — don't accept "we'll figure it out later" for Critical items.

### Step 5: Surface conflicts

If brief + codebase disagree, flag it:
- "Brief says Python/FastAPI but codebase is Node/Fastify — which wins?"
- "Brief says serverless but uses Postgres with connection pooling — these can coexist but you need PgBouncer. Acknowledge?"

### Step 6: Write BACKEND_INTAKE.md

Save to `.backend/<feature>/BACKEND_INTAKE.md`:

**Lineage-only sections:** sections marked with an HTML comment starting `<!-- lineage-only: ... -->` must be emitted ONLY when `.pm/<feature>/PARENT.md` exists. If lineage is absent, omit the entire section (comment and heading). When emitting, delete the HTML comment line — it's an authoring marker, not document content.

```markdown
# Backend Intake: [Feature Name]

**Date:** [today]
**Brief:** BACKEND_BRIEF.md
**Codebase:** [existing / greenfield]

## What's clear (from brief + codebase)

| Concern | Value | Source |
|---------|-------|--------|
| Runtime | Node.js 20 LTS | brief + package.json engines |
| Database | PostgreSQL 16 | brief |
| ... | | |

## Ambiguities resolved in this intake

| Concern | Options considered | Chosen | Rationale |
|---------|-------------------|--------|-----------|
| Auth | roll-your-own JWT / Clerk / Supabase | Clerk | provider handles SSO + passkeys out of the box; 1 engineer saved 2 weeks |
| API style | REST / GraphQL / tRPC | REST | frontend is a marketing SPA, relational queries not needed |

## Decisions deferred to backend-stack

| Concern | Why deferred |
|---------|--------------|
| Exact framework (Fastify vs Hono) | needs one more comparison against existing team familiarity |

## Codebase reality

- Runtime detected: [node 20]
- Framework detected: [none / fastify 4.x]
- Deps matching our expectations: [prisma, zod, vitest]
- Deps NOT matching: [express — legacy, to be removed]
- Migration state: [10 prior migrations, schema stable]

<!-- lineage-only: emit this entire section ONLY when PARENT.md exists; delete this comment on emit -->
## Parent baseline

| Artifact | Path | Key inheritance |
|----------|------|-----------------|
| Backend stack | .backend/<parent-slug>/BACKEND_STACK.md | [one-line summary of inherited runtime/framework/DB/auth] |
| Data model | .backend/<parent-slug>/BACKEND_DATA.md | [one-line summary of inherited entities] |
| API surface | .backend/<parent-slug>/BACKEND_API.md | [one-line summary of inherited endpoints and conventions] |

## Conflicts surfaced

- [Brief says X, codebase has Y → resolution]

## Open questions for backend-stack

- [Anything the grill couldn't resolve — with a note on why]
```

### Step 7: Hand off

Tell the user: "Intake complete. [N] concerns classified, [M] ambiguities resolved, [K] deferred to backend-stack. Next: `backend-stack` will lock in framework/ORM/structure/conventions."

## Rules

- Never re-ask decisions already marked ✅. Wastes the user's time and signals you didn't read the brief.
- For Critical ambiguities (runtime, DB, auth), do not accept "skip for now" — push to a decision or explicitly flag as blocker.
- If the codebase contradicts the brief on a foundational choice, resolve before proceeding — don't let `backend-stack` inherit the ambiguity.
- If the brief is thin (< 30 lines), expect the grill to be longer — that's fine.
- This skill does not make implementation decisions about folder structure or coding conventions. Those belong to `backend-stack`.
- When `.pm/<feature>/PARENT.md` exists, inherit the parent's `BACKEND_STACK.md`, `BACKEND_DATA.md`, and `BACKEND_API.md` as ✅ clear. Never re-grill runtime, framework, DB, ORM, validation, auth strategy, API style, endpoint conventions, or migration history when the parent already resolved them. New concerns introduced by the evolution are still in scope for the grill.
