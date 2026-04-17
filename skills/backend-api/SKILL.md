---
name: backend-api
description: Reads `.backend/<feature>/BACKEND_STACK.md`, `BACKEND_DATA.md`, and `BACKEND_BRIEF.md` and produces `.backend/<feature>/BACKEND_API.md` — full API contract (endpoint table, request/response schemas, auth per route, pagination/filtering conventions, error envelope, versioning, rate limits) plus a serialized contract artifact (OpenAPI YAML, GraphQL SDL, or tRPC router signature) sufficient for frontend codegen. Use when backend-flow invokes it or when user says "design the API", "spec the endpoints". Not for reopening stack decisions or for UI concerns.
---

# Backend API — Contract Specification

Turns the data model and brief into a precise, codegen-ready contract.

## How to run

### Step 1: Load inputs

Read:
- `.backend/<feature>/BACKEND_STACK.md` (framework idiom: REST / GraphQL / tRPC)
- `.backend/<feature>/BACKEND_DATA.md` (entities and fields)
- `.backend/<feature>/BACKEND_BRIEF.md` (API surface hints from PM)

Abort if any are missing.

### Step 2: Build the endpoint table

One row per endpoint. Tie each to a Functional Requirement (FR-XXX) from the PM brief when possible.

| Method | Path | Description | Auth | Maps to FR | Request | Response |
|--------|------|-------------|------|------------|---------|----------|
| POST | /v1/auth/login | Authenticate user | public | FR-001 | `{email, password}` | `{accessToken, refreshToken, user}` |
| GET | /v1/projects | List my projects | user | FR-010 | query: `?cursor&limit&status` | `{items: Project[], nextCursor}` |
| POST | /v1/projects | Create project | user | FR-011 | `{name, slug?}` | `Project` |
| GET | /v1/projects/:id | Get project | user (owner or member) | FR-012 | — | `Project` |
| PATCH | /v1/projects/:id | Update project | user (owner) | FR-013 | `{name?, archived?}` | `Project` |
| DELETE | /v1/projects/:id | Delete project | user (owner) | FR-014 | — | `204 No Content` |
| GET | /v1/projects/:id/tasks | List tasks | user | FR-020 | query: `?cursor&limit&status&assignee` | `{items: Task[], nextCursor}` |

### Step 3: Define the error envelope

Every error response has the same shape:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "email is required",
    "details": [{ "field": "email", "code": "required" }]
  },
  "requestId": "req_abc123"
}
```

Error code taxonomy (stable, versioned):
- `VALIDATION_ERROR` — 400
- `UNAUTHORIZED` — 401
- `FORBIDDEN` — 403
- `NOT_FOUND` — 404
- `CONFLICT` — 409
- `RATE_LIMITED` — 429
- `INTERNAL_ERROR` — 500

### Step 4: Cross-cutting conventions

**Pagination:** cursor-based. `?cursor=<opaque>&limit=20`. Response: `{items, nextCursor}`. No offset pagination unless a specific route justifies it.

**Filtering:** simple query params (`?status=open&assignee=u_abc`). No query-param DSL like `?filter[status]=open`. If complex filtering is needed, move to `POST /search`.

**Sorting:** `?sort=createdAt&order=desc`. Single field. Multi-field sort only when a route demands it.

**Field selection:** none by default. Add `?fields=id,name,status` only when responses are heavy. Use sparingly.

**Versioning:** path-based, `/v1/...`. Breaking changes bump to `/v2`. Old versions supported for ≥6 months.

**Rate limiting:** per-user for authenticated routes (100 req/min). Per-IP for public routes (20 req/min).

### Step 5: Security per endpoint

For each route, declare:
- Auth required? (public / user / role: admin)
- Input validation schema name (from validation library)
- Rate limit bucket (default-user / default-public / custom)
- Response field redaction (e.g., never return `password_hash`, `refresh_token`)

### Step 6: Serialize the contract

Pick the format based on stack:

**REST → OpenAPI 3.1 YAML**
**GraphQL → SDL (.graphql)**
**tRPC → router signature (TypeScript)**

Emit enough to drive frontend codegen. Do not write the YAML/SDL into the user's source tree from this skill — embed as a fenced code block in BACKEND_API.md.

### Step 7: Write BACKEND_API.md

Save to `.backend/<feature>/BACKEND_API.md`. Structure:

````markdown
# Backend API: [Feature Name]

**Stack:** BACKEND_STACK.md
**Data:** BACKEND_DATA.md
**Style:** REST + OpenAPI 3.1
**Base URL:** `https://api.example.com/v1`

## Endpoint table

[From Step 2]

## Error envelope

[Shape + codes]

## Conventions

### Pagination
Cursor-based. [Details from Step 4]

### Filtering / sorting / fields / versioning / rate limits
[Details]

## Security per endpoint

| Route | Auth | Validation schema | Rate limit bucket |
|-------|------|-------------------|-------------------|
| POST /v1/auth/login | public | `LoginSchema` | default-public |
| GET /v1/projects | user | — | default-user |
| POST /v1/projects | user | `CreateProjectSchema` | default-user |
| PATCH /v1/projects/:id | user (owner) | `UpdateProjectSchema` | default-user |

## OpenAPI

```yaml
openapi: 3.1.0
info:
  title: [Feature] API
  version: 1.0.0
servers:
  - url: https://api.example.com/v1
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    Error:
      type: object
      required: [error, requestId]
      properties:
        error:
          type: object
          required: [code, message]
          properties:
            code: { type: string }
            message: { type: string }
            details: { type: array, items: { type: object } }
        requestId: { type: string }
    Project:
      type: object
      required: [id, name, slug, ownerId, createdAt]
      properties:
        id: { type: string, format: uuid }
        name: { type: string }
        slug: { type: string }
        ownerId: { type: string, format: uuid }
        archivedAt: { type: string, format: date-time, nullable: true }
        createdAt: { type: string, format: date-time }
paths:
  /auth/login:
    post:
      summary: Authenticate user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email: { type: string, format: email }
                password: { type: string, minLength: 8 }
      responses:
        '200':
          description: Success
          content:
            application/json:
              schema:
                type: object
                properties:
                  accessToken: { type: string }
                  refreshToken: { type: string }
                  user: { $ref: '#/components/schemas/User' }
        '401':
          description: Invalid credentials
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }
  # ... other paths
```

## tRPC alternative (if applicable)

```ts
// Shape reference
export const appRouter = router({
  auth: router({
    login: publicProcedure
      .input(z.object({ email: z.string().email(), password: z.string().min(8) }))
      .mutation(async ({ input }) => { /* ... */ }),
  }),
  projects: router({
    list: protectedProcedure
      .input(z.object({ cursor: z.string().optional(), limit: z.number().default(20) }))
      .query(async ({ input, ctx }) => { /* ... */ }),
    create: protectedProcedure
      .input(z.object({ name: z.string(), slug: z.string().optional() }))
      .mutation(async ({ input, ctx }) => { /* ... */ }),
  }),
});

export type AppRouter = typeof appRouter;
```
````

### Step 8: Hand off

Tell the user: "API contract specced. [N] endpoints, [K] shared schemas. Frontend can generate types from this. Next: `backend-tasks` breaks the implementation into vertical slices."

## Rules

- Every endpoint maps to a declared FR or is flagged as "infrastructure/scaffold" in the description.
- Error envelope is consistent across every endpoint. No ad-hoc error shapes per route.
- Pagination defaults to cursor — only switch to offset with an explicit reason recorded in the decision log.
- Never return secrets, hashes, internal IDs that leak structure. Redaction is part of the contract, not an afterthought.
- The serialized contract (OpenAPI / SDL / tRPC) must be complete enough for frontend codegen — no TODOs embedded.
