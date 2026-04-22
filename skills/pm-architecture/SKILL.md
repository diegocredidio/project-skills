---
name: pm-architecture
description: Reads prior PM artifacts from `.pm/<feature>/PRD.md` and produces the technical architecture at `.pm/<feature>/ARCHITECTURE.md` — system design, stack choices, API boundaries, data model, infrastructure. Use when user says "define the architecture", "tech stack", "system design", "how should we build this", or when the pm-flow orchestrator triggers this step. Not for open-ended architecture consulting or technology comparisons disconnected from a named project.
---

# PM Architecture — Technical Architecture Definition

Translate the PRD into concrete technical decisions. This is where product requirements become system design.

## How to run

### Step 1: Gather inputs

Read these in order:
1. `.pm/<feature-name>/PRD.md` — the source of truth for what we're building
2. `.pm/<feature-name>/GRILL_SUMMARY.md` — for technical constraints discussed
3. Explore the existing codebase thoroughly:
   - `package.json`, `requirements.txt`, `go.mod`, `Cargo.toml` — what's the stack?
   - `docker-compose.yml`, `Dockerfile`, `*.tf`, `.github/workflows/` — infrastructure
   - Database migrations, schema files, ORM models — data model
   - API routes, controllers, handlers — existing API surface
   - `.env.example`, config files — environment and integrations
   - Component library, design system files — frontend patterns
4. **Lineage check:** if `.pm/<feature-name>/PARENT.md` exists:
   - Read the parent slug from `PARENT.md`'s H1 (`# Parent: <parent-slug>`).
   - Read `.pm/<parent-slug>/ARCHITECTURE.md` — treat the parent's stack decisions, API conventions, and data model as the baseline. Do NOT re-derive them.
   - The child's architecture is a **delta document** over the parent's. In the output, mark every section as `inherited`, `extended`, or `new` (see Step 3 template).

### Step 1.5: Read PROJECT_PROFILE.md

Read `.pm/<feature-name>/PROJECT_PROFILE.md` to get `testingRigor` (and `designMode`, if present). This value drives the `## Testability` section emitted in Step 3.

If the file is missing, prompt once:

> "Este projeto não tem PROJECT_PROFILE.md completo. Modo de design: **shadcn-theme** ou **custom-system**? Rigor de teste: **mvp** ou **full**? (Respostas gravam em `.pm/<feature-name>/PROJECT_PROFILE.md` — afetam design-tokens, design-components, qa-strategy, qa-review.)"

Write the answers to `.pm/<feature-name>/PROJECT_PROFILE.md`, then proceed.

### Step 2: Ask architecture-specific questions

Based on what you found (or didn't find), ask:
- "Is this a new project or adding to an existing system?"
- "Are there specific technology choices already made or required?"
- "What's the deployment target? (Cloud provider, serverless, containers, etc.)"
- "Do you need to support real-time features? (WebSockets, SSE, polling)"
- "What's the authentication strategy? (Existing auth, new auth, third-party)"
- "What external services are involved?" (Payment, email, storage, AI, etc.)

If the codebase already answers these, state what you found instead of asking.

### Step 3: Write the architecture document

```markdown
# Technical Architecture: [Feature/Project Name]

**Based on:** PRD v[X]
**Date:** [Today]

---

## 1. System Overview

[2-3 sentence summary of the technical approach.
A high-level diagram description — what talks to what.]

### System diagram (textual)

```
[Client] → [API Gateway / BFF] → [Service A]
                                → [Service B]
                                → [Database]
                                → [External: Payment API]
```

<!-- lineage-only: emit this subsection ONLY when PARENT.md exists; delete this comment on emit -->
### Lineage

This architecture extends `<parent-slug>`'s baseline. Summary of deltas:

- **Reused wholesale:** [list subsystems not touched — e.g., existing auth tables, session management]
- **Extended:** [list subsystems the child adds to — e.g., new `[child_table]` table, new `/[path]/*` endpoints]
- **Modified:** [list parent behaviors the child changes]

See `.pm/<parent-slug>/ARCHITECTURE.md` for the full baseline.

## 2. Stack Decisions

**Inherited vs. new:** when `.pm/<feature-name>/PARENT.md` exists, include the `Status` column below (values: `inherited`, `extended`, `new`) and flag each layer accordingly. When standalone, omit the `Status` column entirely.

| Layer | Choice | Status | Rationale |
|-------|--------|--------|-----------|
| Frontend | [Framework] | [inherited\|extended\|new] | [Why — if inherited, cite parent; if extended/new, justify the delta] |
| Backend | [Language/Framework] | [inherited\|extended\|new] | [Why] |
| Database | [DB type and name] | [inherited\|extended\|new] | [Why] |
| Auth | [Strategy] | [inherited\|extended\|new] | [Why] |
| Hosting | [Platform] | [inherited\|extended\|new] | [Why] |
| CI/CD | [Tool] | [inherited\|extended\|new] | [Why] |

### Existing stack (if applicable)
[What's already in place and what we're adding to it]

## 3. API Design

### Endpoints

| Method | Path | Description | Auth | Maps to FR |
|--------|------|-------------|------|------------|
| POST | /api/v1/[resource] | [Description] | [Yes/No] | FR-001 |
| GET | /api/v1/[resource]/:id | [Description] | [Yes/No] | FR-002 |
...

### API conventions
[REST vs GraphQL, naming conventions, error format, pagination strategy]

## 4. Data Model

### Entities

#### [Entity Name]
| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK | |
| [field] | [type] | [constraints] | [notes] |

### Relationships
[Entity A] → has many → [Entity B]
[Entity B] → belongs to → [Entity A]

### Indexes
[Key indexes for query performance]

## 5. Frontend Architecture

### Pages / Routes
| Route | Component | Description | Maps to FR |
|-------|-----------|-------------|------------|
| /[path] | [Component] | [Description] | FR-XXX |

### State management
[Strategy: local state, context, global store, server state]

### Component structure
[High-level component tree or module organization]

## 6. Infrastructure

### Environments
[Dev, staging, production — what's different between them]

### Deployment
[How code gets from PR to production]

### Monitoring
[Logging, error tracking, performance monitoring]

### Security considerations
[Authentication flow, data encryption, API rate limiting, CORS]

## 7. Integration Points

| System | Purpose | Protocol | Auth method |
|--------|---------|----------|-------------|
| [External service] | [Why] | [REST/gRPC/etc] | [API key/OAuth/etc] |

## 8. Testability

**testingRigor:** [mvp | full — from PROJECT_PROFILE.md]

For `full` rigor, architecture MUST provide:
- Seams for mocking external dependencies (interfaces / ports / adapters)
- A dedicated test database or a migration-safe test schema
- API designed for contract testing (versioned, deterministic error shapes)
- Observability hooks accessible from tests (log capture, trace IDs)

For `mvp` rigor, these are nice-to-haves; defer until scope grows.

Flag any architectural decisions above that would block the chosen rigor level.

## 9. Technical Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| [Risk] | [H/M/L] | [H/M/L] | [Plan] |

## 10. Decision Log

| # | Decision | Options considered | Chosen | Rationale |
|---|----------|-------------------|--------|-----------|
| 1 | [Decision] | [A, B, C] | [B] | [Why B] |
```

### Step 4: Review with user

Present and ask:
- "Does the stack align with your team's skills?"
- "Any integration I'm missing?"
- "Is the data model right for your query patterns?"
- "Are the API boundaries where you'd expect them?"

### Step 5: Save

Save to `.pm/<feature-name>/ARCHITECTURE.md`.

## Rules

- If the project has an existing codebase, RESPECT IT. Don't suggest a new framework when one exists.
- Every API endpoint must trace back to a functional requirement (FR-XXX) from the PRD
- Every architectural decision needs a rationale — "because it's popular" is not a rationale
- Flag technical risks explicitly — don't hide complexity
- If the user's constraints conflict (e.g., "real-time + serverless + no budget"), surface the tension and propose trade-offs
- Keep the data model focused on this feature — don't redesign the entire database
- When `.pm/<feature-name>/PARENT.md` exists, the child's architecture is a delta document. Inherit the parent's stack verbatim unless there is an explicit reason to change — and if you change, document the reason in the Decision Log table. Flag every decision as inherited / extended / new.
- Never silently diverge from the parent's stack. A different ORM, language, or auth provider is a Decision, not a detail.
