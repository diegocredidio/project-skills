# project-skills

**35 skills for Claude Code that take a project from a vague idea to a Jira-backed implementation plan across PM, design, backend, and frontend — with room to re-run any one phase when a decision changes.**

Inspired by [Julian Oczkowski's designer-skills](https://github.com/julianoczkowski/designer-skills) and [Matt Pocock's skills](https://github.com/mattpocock/skills) — rewritten from scratch, no external runtime dependencies.

---

## What this gives you

Five families of skills, each a re-invocable piece of a larger flow:

```
pm-flow
  pm-intake? → pm-grill → pm-prd → pm-architecture
  → pm-workstreams → pm-tasks → pm-review → pm-handoff
                                               ↓
      ┌────────────────────┬───────────────────┴────────────────┐
      ↓                    ↓                                    ↓
 design-flow         backend-flow                        frontend-flow
 (Path A direct)                                  (depends on design outputs)
      ↓                    ↓                                    ↓
 design-brief-intake  backend-brief-intake               frontend-brief-intake
      ↓                    ↓                                    ↓
 design-ia           backend-stack                        frontend-stack
      ↓                    ↓                                    ↓
 design-tokens       backend-data                         frontend-routing
      ↓                    ↓                                    ↓
 design-components   backend-api                          frontend-components
      ↓                    ↓                                    ↓
 design-tasks        backend-tasks                        frontend-tasks
      ↓                    ↓                                    ↓
 (build)             (build)                              (build)
      ↓                    ↓                                    ↓
 design-review       backend-review                        frontend-review
```

Then, whenever you're ready to put the plan on a board:

```
kanban-flow
  kanban-sync     (TASKS.md files → Jira: Epic + Stories + Tasks + dependency links)
  kanban-status   (board → Kanban flow report: WIP, blockers, cycle time, stale cards)
  kanban-pickup   (dev picks next card → transition to In Progress + load local spec context)
```

## Install

Two ways, same repo, no duplication. Pick what fits your setup.

### Option 1 — Native Claude Code plugin (recommended)

```
/plugin marketplace add diegocredidio/project-skills
/plugin install project-skills@project-skills
```

Skills appear under the `project-skills:` namespace (`project-skills:pm-flow`, `project-skills:design-tokens`, etc.).

### Option 2 — Vercel skills CLI (multi-editor)

```
npx skills add diegocredidio/project-skills
```

Works with Claude Code, Cursor, Windsurf, OpenCode, and any agent that reads `.claude/skills/`. Skills appear by bare name (`pm-flow`, `design-tokens`).

Both modes read the same `skills/` tree — pick whichever matches your workflow.

## Artifacts on disk

Every skill reads from and writes to `.<discipline>/<feature>/` in the target repository:

```
.pm/<feature>/
  ├── INTAKE.md              (pm-intake)
  ├── GRILL_SUMMARY.md       (pm-grill)
  ├── PRD.md                 (pm-prd)
  ├── ARCHITECTURE.md        (pm-architecture)
  ├── WORKSTREAMS.md         (pm-workstreams)
  ├── TASKS.md               (pm-tasks)
  ├── REVIEW.md              (pm-review)
  └── JIRA_MAP.md            (kanban-sync idempotency map)

.design/<feature>/
  ├── DESIGN_GRILL.md        (design-grill, Path B only)
  ├── DESIGN_BRIEF.md        (pm-handoff; enriched by design-brief-intake)
  ├── CODEBASE_AUDIT.md      (design-brief-intake)
  ├── IA.md                  (design-ia)
  ├── TOKENS.md              (design-tokens)
  ├── COMPONENT_SPECS.md     (design-components)
  ├── DESIGN_TASKS.md        (design-tasks)
  ├── DESIGN_REVIEW.md       (design-review)
  └── screenshots/           (design-review evidence)

.backend/<feature>/
  ├── BACKEND_BRIEF.md       (pm-handoff)
  ├── BACKEND_INTAKE.md      (backend-brief-intake)
  ├── BACKEND_STACK.md       (backend-stack)
  ├── BACKEND_DATA.md        (backend-data)
  ├── BACKEND_API.md         (backend-api)
  ├── BACKEND_TASKS.md       (backend-tasks)
  └── BACKEND_REVIEW.md      (backend-review)

.frontend/<feature>/
  ├── FRONTEND_BRIEF.md      (pm-handoff)
  ├── FRONTEND_INTAKE.md     (frontend-brief-intake)
  ├── FRONTEND_STACK.md      (frontend-stack)
  ├── FRONTEND_ROUTES.md     (frontend-routing)
  ├── COMPONENT_PLAN.md      (frontend-components)
  ├── FRONTEND_TASKS.md      (frontend-tasks)
  └── FRONTEND_REVIEW.md     (frontend-review)

.pm/jira-config.md            (kanban-flow, first-run setup)
```

Every artifact is plain markdown. You can edit them by hand, commit them, share them. When a decision changes, re-run the relevant skill and the downstream artifacts update from there.

## Principles

### Process over prompting

"Build me an app" produces something that looks like software but isn't grounded in decisions. These skills encode the process so every choice is intentional, traceable, and testable.

### Vertical slices

Every `*-tasks` skill outputs vertical slices — a thin end-to-end increment (migration + service + route + test; or route + component + data + states + tests). You can ship incrementally and get feedback early instead of "finish all APIs, then all UI".

### Codebase-aware, every step

Every skill that touches output audits the repo first: existing routes, existing tokens, existing schemas, existing components. No regeneration of what already exists. No cargo-culted structure.

### Persistence you can trust

All state lives on disk under `.<discipline>/<feature>/`. Resume across sessions, diff in git, hand off between teammates. Skills never rely on conversation history alone.

### Jira is a thin layer, not the source of truth

`kanban-sync` is idempotent. Source of truth is the local markdown. Jira reflects reality; `JIRA_MAP.md` keeps them linked. Delete a task locally → it becomes `Cancelled` in Jira (preserves history).

## Scope vs. superpowers

If you use the [superpowers](https://github.com/obra/superpowers) skill pack, these are complementary, not competing:

- `superpowers:brainstorming` — for **open-ended ideation**, before a scope exists. Use this first when you have a fuzzy idea.
- `superpowers:writing-plans` — for **generic multi-step task authoring**, when the work is small and doesn't need the full PM/Design/Backend/Frontend ladder.
- `superpowers:executing-plans` — for executing an existing plan with review checkpoints.
- **`project-skills:pm-flow`** — fills the gap between brainstorming and plan execution for **named projects or features** with a real PRD / architecture / workstream decomposition.
- **`project-skills:design-flow`, `backend-flow`, `frontend-flow`** — discipline-specific decomposition when a project has artifacts ready to hand off.
- **`project-skills:kanban-*`** — project management layer once the plan exists.

Every skill in this pack has an anti-collision clause in its description: if the subject is "open-ended", `superpowers:brainstorming` wins. If the subject is "plan this named feature end-to-end", `project-skills:pm-flow` wins.

## Requirements

- **Claude Code** (plugin mode) or a supported editor (Vercel CLI mode).
- For `design-review` and `frontend-review`: **Playwright MCP** server — screenshots are mandatory; skills abort without it.
- For `kanban-*`: **Atlassian MCP** server with access to an existing Jira project. Skills do not create Jira projects.

## Credits

- Process-as-skills concept from Julian Oczkowski's [designer-skills](https://github.com/julianoczkowski/designer-skills) — inspiration, no code reused.
- Grill-me pattern inspired by [Matt Pocock's skills](https://github.com/mattpocock/skills).
- "Design of Design" tree-of-decisions framing from Frederick P. Brooks Jr.

## License

MIT. See [LICENSE](./LICENSE).
