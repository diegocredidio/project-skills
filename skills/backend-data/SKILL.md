---
name: backend-data
description: Reads `.backend/<feature>/BACKEND_STACK.md` (for ORM / DB choice) and `BACKEND_BRIEF.md` (for entities named by PM) and produces `.backend/<feature>/BACKEND_DATA.md` — concrete data model with entities, fields, constraints, relationships, indexes, and migration strategy. Emits ORM-specific snippets (schema.prisma / SQLAlchemy model / GORM struct) as shape references. Use when backend-flow invokes it or when user says "define the data model", "design the schema". Not for schema-less modeling discussions or for changing the database choice (that's backend-stack).
---

# Backend Data — Concrete Data Model

Translates the entity hints in the brief into a schema the team can implement.

## How to run

### Step 1: Load inputs

Read:
- `.backend/<feature>/BACKEND_STACK.md` (for ORM idiom)
- `.backend/<feature>/BACKEND_BRIEF.md` (data model section — entities named by PM)
- Optionally `.backend/<feature>/BACKEND_INTAKE.md` (for conflicts already resolved)

If either STACK or BRIEF is missing, abort and direct the user to run the missing predecessor.

### Step 2: Expand each entity into a field table

For every entity named in the brief, produce a field table:

| Field | Type | Constraints | Default | Notes |
|-------|------|-------------|---------|-------|
| id | uuid | PK | `gen_random_uuid()` | surrogate key |
| email | text | unique, not null | — | lowercase on write |
| created_at | timestamptz | not null | `now()` | |
| deleted_at | timestamptz | nullable | null | soft delete |

Pick types that match the DB engine chosen in BACKEND_STACK.md (e.g., `uuid` for Postgres, `TEXT`/`INTEGER` for SQLite, etc.).

### Step 3: Define relationships explicitly

For each relationship, say:
- Direction (User has many Projects / Project belongs to one User)
- Cardinality (1:1 / 1:N / N:M)
- Join table name (for N:M)
- ON DELETE behavior (cascade / restrict / set null)

### Step 4: Define indexes from query patterns

Walk the likely API queries (from BACKEND_BRIEF or from common sense):
- "List projects by owner" → index on `projects(owner_id, created_at DESC)`
- "Look up user by email" → unique index on `users(email)` — already covered by the unique constraint
- "Find tasks by project with status filter" → composite index on `tasks(project_id, status)`

Prefer composite indexes where queries predictably filter together. Flag any table where index choice is ambiguous pending real query logs.

### Step 5: Migration strategy

Pick one (should match BACKEND_STACK's ORM):
- Prisma Migrate — `prisma migrate dev` / `prisma migrate deploy`
- Alembic — `alembic revision --autogenerate` / `alembic upgrade head`
- golang-migrate — numbered SQL files up/down
- Rails ActiveRecord — `bin/rails db:migrate`

Declare:
- Naming convention (`YYYYMMDDHHMM_description.sql` or equivalent)
- Rollback policy (reversible? destructive changes require explicit review?)
- Seed data approach (separate seed script; not run in production)

### Step 6: Write BACKEND_DATA.md

Save to `.backend/<feature>/BACKEND_DATA.md`:

````markdown
# Backend Data Model: [Feature Name]

**Stack:** BACKEND_STACK.md
**Database:** [PostgreSQL 16]
**ORM:** [Prisma 5]

## Entities

### User

| Field | Type | Constraints | Default | Notes |
|-------|------|-------------|---------|-------|
| id | uuid | PK | gen_random_uuid() | |
| email | text | unique, not null | — | stored lowercase |
| name | text | not null | — | |
| created_at | timestamptz | not null | now() | |
| updated_at | timestamptz | not null | now() | trigger to bump |

### Project

| Field | Type | Constraints | Default | Notes |
|-------|------|-------------|---------|-------|
| id | uuid | PK | gen_random_uuid() | |
| owner_id | uuid | FK → users.id, ON DELETE CASCADE, not null | — | |
| name | text | not null | — | |
| slug | text | unique per owner, not null | — | composite unique (owner_id, slug) |
| archived_at | timestamptz | nullable | null | soft archive |
| created_at | timestamptz | not null | now() | |

### Task

| Field | Type | Constraints | Default | Notes |
|-------|------|-------------|---------|-------|
| id | uuid | PK | gen_random_uuid() | |
| project_id | uuid | FK → projects.id, ON DELETE CASCADE, not null | — | |
| title | text | not null | — | |
| status | text | enum(open/in_progress/done), not null | 'open' | |
| assignee_id | uuid | FK → users.id, ON DELETE SET NULL, nullable | null | |

## Relationships

- User 1:N Project (via `projects.owner_id`)
- Project 1:N Task (via `tasks.project_id`)
- User 0:N Task (assignee, nullable — via `tasks.assignee_id`)

## Indexes

| Table | Index | Reason |
|-------|-------|--------|
| users | unique(email) | login lookup + constraint |
| projects | composite(owner_id, created_at DESC) | list projects by owner, newest first |
| projects | unique(owner_id, slug) | slug uniqueness per owner |
| tasks | composite(project_id, status) | filter tasks by project and status |
| tasks | (assignee_id) | "my tasks" query |

## Migration strategy

- Tool: Prisma Migrate
- Naming: timestamp-prefixed, e.g. `20260417140530_add_tasks_table`
- Rollback: preview-environment-only. Production migrations must be additive (add columns nullable; backfill; flip not-null in follow-up migration).
- Seeds: `prisma/seed.ts`, never run in production.

## ORM snippets

### schema.prisma (primary)

```prisma
model User {
  id         String    @id @default(uuid()) @db.Uuid
  email      String    @unique @db.Text
  name       String    @db.Text
  projects   Project[]
  createdAt  DateTime  @default(now()) @map("created_at")
  updatedAt  DateTime  @updatedAt      @map("updated_at")
  @@map("users")
}

model Project {
  id         String    @id @default(uuid()) @db.Uuid
  ownerId    String    @map("owner_id") @db.Uuid
  owner      User      @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  name       String    @db.Text
  slug       String    @db.Text
  tasks      Task[]
  archivedAt DateTime? @map("archived_at")
  createdAt  DateTime  @default(now()) @map("created_at")
  @@unique([ownerId, slug])
  @@index([ownerId, createdAt])
  @@map("projects")
}

model Task {
  id         String   @id @default(uuid()) @db.Uuid
  projectId  String   @map("project_id") @db.Uuid
  project    Project  @relation(fields: [projectId], references: [id], onDelete: Cascade)
  title      String   @db.Text
  status     String   @default("open") @db.Text
  assigneeId String?  @map("assignee_id") @db.Uuid
  @@index([projectId, status])
  @@index([assigneeId])
  @@map("tasks")
}
```

### Alternative: SQLAlchemy (if applicable)

```py
# Shape reference only — actual model in app/models/
class User(Base):
    __tablename__ = 'users'
    id = Column(UUID, primary_key=True, default=uuid4)
    email = Column(Text, unique=True, nullable=False)
    # ...
```
````

### Step 7: Hand off

Tell the user: "Data model specced. [N] entities, [M] relationships, [K] indexes. Next: `backend-api` translates this into endpoint contracts."

## Rules

- Types must match the chosen DB engine — do not suggest `uuid` for SQLite, or `jsonb` for MySQL (< 8.0.17).
- Every foreign key needs an explicit ON DELETE behavior. Never leave it to the ORM default.
- Indexes must be justified by a query pattern. Don't sprinkle indexes "just in case".
- Migration strategy must commit to a rollback policy — vague "we'll figure it out" is not acceptable for production.
- ORM snippets are **shape references**, not implementation. They go in the markdown; actual code is written in the implementation step.
