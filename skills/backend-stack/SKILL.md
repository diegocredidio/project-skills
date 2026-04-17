---
name: backend-stack
description: Reads `.backend/<feature>/BACKEND_INTAKE.md` and produces `.backend/<feature>/BACKEND_STACK.md` — the canonical stack decision document (runtime, framework, database, ORM, validation, auth, testing, hosting), project folder structure, coding conventions, environment variables, and decision log. Every row has a rationale tied to the intake. Use when backend-flow invokes it or when user says "define the backend stack", "decide the framework". Not for reopening PM-level architecture or for language debates disconnected from a named project.
---

# Backend Stack — Decision Document

Locks in every backend technology and pattern choice for a specific project.

## How to run

### Step 1: Load intake

Read `.backend/<feature>/BACKEND_INTAKE.md`. Extract resolved decisions, codebase constraints, and deferred questions.

If BACKEND_INTAKE.md does not exist, abort. Tell the user: "Run `backend-brief-intake` first — I need the gap-fill grill results."

### Step 2: Fill the stack decision table

For each row, either confirm what intake resolved or decide now if deferred. Every choice needs a rationale.

| Concern | Typical options | Default |
|---------|-----------------|---------|
| Runtime | Node 20 LTS / Python 3.12 / Go 1.22 | Node 20 LTS |
| Framework | Fastify / Hono / Express / FastAPI / Gin | Fastify (Node) |
| Database | PostgreSQL 16 / MySQL 8 / SQLite | PostgreSQL 16 |
| ORM / query | Prisma / Drizzle / SQLAlchemy / GORM / raw SQL | Prisma (Node) |
| Validation | Zod / Valibot / Pydantic / go-playground/validator | Zod |
| Auth | Clerk / Auth0 / Supabase / roll-your-own (JWT+bcrypt) | Clerk (provider) |
| Testing | Vitest + Supertest / Jest / pytest / go test | Vitest + Supertest |
| Hosting | Railway / Fly.io / Vercel / AWS / container + PaaS | Railway |

### Step 3: Render the folder structure

Be concrete. Adapt to the chosen stack. Examples below — use them as starting points, not verbatim templates.

**Node.js / Fastify / Prisma:**

```
src/
├── routes/          # one file per resource
├── services/        # business logic, no HTTP concerns
├── repositories/    # data access, wraps Prisma client
├── middleware/      # auth, rate-limit, error handler
├── plugins/         # Fastify plugins (db, cors, auth)
├── schemas/         # Zod schemas for request/response validation
├── jobs/            # background jobs
└── utils/           # pure helpers
prisma/
├── schema.prisma
└── migrations/
tests/
├── unit/
├── integration/
└── fixtures/
```

**Python / FastAPI / SQLAlchemy:**

```
app/
├── api/v1/
│   ├── routes/
│   └── deps.py
├── core/            # config, security, db session
├── models/          # SQLAlchemy ORM models
├── schemas/         # Pydantic request/response
├── services/
├── repositories/
└── tasks/           # Celery/APScheduler
alembic/
tests/
```

**Go / Gin:**

```
cmd/
└── server/main.go
internal/
├── handler/         # Gin handlers
├── service/
├── repository/
├── middleware/
└── domain/          # Entity structs
migrations/
```

### Step 4: Define coding conventions

Concretely — two developers must reach the same conclusion independently.

**Error propagation:**
- Repository returns `Result<T, RepoError>` / raises `RepositoryError`
- Service catches and maps to domain errors
- Route maps domain errors to HTTP status codes via a central error handler
- Response envelope: `{ error: { code, message, details? } }`

**Input validation:**
- Always at the route boundary, before reaching service
- Use [chosen validation library] schemas colocated with the route file
- Reject with 400 + error envelope; never let invalid input reach service layer

**Logging:**
- Structured JSON
- Required fields on every log entry: `requestId`, `userId` (if auth'd), `timestamp`, `level`, `message`
- Use correlation ids propagated through context

**Testing strategy:**
- Unit: services and repositories with mocked deps
- Integration: full HTTP → DB round-trip, no mocks
- Use a real test database (not mocks) for integration — spin up via docker-compose
- Do NOT test framework internals or generated ORM code

**Data access:**
- All DB access through repository layer — no raw queries in services
- Transactions explicit — declared at service layer, passed to repository methods as a context
- N+1 prevention: repositories accept `include`/`with` params; eager-load by default for user-facing queries

**Auth pattern:**
- Middleware attaches `req.user` after validating token
- Protected routes declare `auth: true` via plugin / decorator
- Permission checks in service layer (not middleware) — the service owns the rule

### Step 5: Environment variables

List every env var with description. No values — just keys and purpose.

| Variable | Required | Description | Example format |
|----------|----------|-------------|----------------|
| DATABASE_URL | yes | PostgreSQL connection | `postgresql://user:pass@host:5432/db` |
| JWT_SECRET | yes | Signing key (≥32 chars) | random string |
| CLERK_SECRET_KEY | yes | Server-side auth key | from Clerk dashboard |
| LOG_LEVEL | no | Default `info` | `debug` / `info` / `warn` / `error` |

### Step 6: Write BACKEND_STACK.md

Save to `.backend/<feature>/BACKEND_STACK.md`. Structure:

```markdown
# Backend Stack: [Feature Name]

**Intake:** BACKEND_INTAKE.md

## Stack decisions

| Concern | Choice | Version | Rationale |
|---------|--------|---------|-----------|
| Runtime | Node.js | 20 LTS | Team familiarity; long-running server |
| Framework | Fastify | 4.x | Plugin ecosystem; TypeScript-first; faster than Express |
| Database | PostgreSQL | 16 | Intake resolved: relational model fits |
| ORM | Prisma | 5.x | Migration DX; type safety; Node ecosystem standard |
| Validation | Zod | 3.x | Inferable TS types; shared with frontend |
| Auth | Clerk | — | Intake resolved: provider over roll-your-own |
| Testing | Vitest + Supertest | — | Faster than Jest; ESM-native |
| Hosting | Railway | — | Postgres + app in one dashboard; deploy from git |

## Project structure

[Tree from Step 3]

## Coding conventions

### Error propagation
[Full detail from Step 4]

### Input validation
[...]

### Logging
[...]

### Testing strategy
[...]

### Data access
[...]

### Auth pattern
[...]

## Environment variables

| Variable | Required | Description | Example format |
|----------|----------|-------------|----------------|
| ... | | | |

## Local development setup

1. `git clone` + `npm ci`
2. `cp .env.example .env` and fill values
3. `docker-compose up -d postgres`
4. `npx prisma migrate dev`
5. `npm run dev`

## Decision log

| # | Decision | Options considered | Chosen | Rationale |
|---|----------|-------------------|--------|-----------|
| 1 | Framework | Fastify / Hono / Express | Fastify | plugin ecosystem + team XP |
| 2 | Auth | Clerk / Auth0 / roll-own | Clerk | saves 2 weeks |
| ... | | | | |
```

### Step 7: Hand off

Tell the user: "Stack locked. Next: `backend-data` takes the entity list from the brief and produces the concrete schema + migration strategy."

## Rules

- Every row in the stack table needs a rationale. "It's popular" is not enough — tie it to this project's constraints from the intake.
- If an existing backend exists, extend it — don't propose a different stack unless the intake explicitly resolves to migrate.
- Folder structure must be concrete (real folder names this project will use), not a generic template.
- Coding conventions must be specific enough that two devs implement the same pattern independently. If vague, tighten or drop.
- This skill does not generate code or scaffold — only decisions, structure, and conventions.
