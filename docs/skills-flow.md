# Skills flow

All 35 skills, their artifacts, and the cross-cutting `PROJECT_PROFILE.md` contract rendered as mermaid diagrams. For a terminal-friendly ASCII view, see the main `README.md`.

---

## Top-level orchestration

```mermaid
flowchart TD
    Start([Idea or PRD draft]) --> PM[pm-flow]
    PM -->|pm-handoff| Design[design-flow]
    PM -->|pm-handoff| Backend[backend-flow]
    PM -->|pm-handoff| Frontend[frontend-flow]
    Design -.specs upstream.-> Frontend
    Backend -.API contracts.-> Frontend
    Design --> Build1[implementação]
    Backend --> Build2[implementação]
    Frontend --> Build3[implementação]
    Build1 --> DReview[design-review]
    Build2 --> BReview[backend-review]
    Build3 --> FReview[frontend-review]
    PM -.TASKS.md.-> Kanban[kanban-flow]
    Design -.DESIGN_TASKS.md.-> Kanban
    Backend -.BACKEND_TASKS.md.-> Kanban
    Frontend -.FRONTEND_TASKS.md.-> Kanban
```

---

## PM flow

```mermaid
flowchart LR
    Intake["pm-intake<br/>opcional (Path B)"] --> Grill["pm-grill<br/>inclui Delivery profile branch"]
    Start([scratch / Path A]) --> Grill
    Grill --> PRD[pm-prd]
    PRD --> Arch[pm-architecture]
    Arch --> WS[pm-workstreams]
    WS --> Tasks[pm-tasks]
    Tasks --> Review[pm-review]
    Review --> Handoff[pm-handoff]
    Grill -.escreve.-> PP[("PROJECT_PROFILE.md<br/>designMode + uiFramework")]
    Handoff -.lê.-> PP
    Handoff --> DesignFlow[design-flow]
    Handoff --> BackendFlow[backend-flow]
    Handoff --> FrontendFlow[frontend-flow]
```

---

## Design flow (com designMode branching)

```mermaid
flowchart TD
    Flow["design-flow<br/>lê PROJECT_PROFILE.md<br/>(prompt migração se faltar)"] --> Brief[design-brief-intake]
    Brief --> IA[design-ia]
    IA --> Mode{designMode?}
    Mode -->|shadcn-theme| TokensSlim["design-tokens<br/>tema slim: HSL vars + radius + font<br/>via tweakcn ou preset shadcn"]
    Mode -->|custom-system| TokensFull["design-tokens<br/>sistema completo<br/>spacing + typography + motion + z-index"]
    TokensSlim --> CompScreen["design-components<br/>screen-centric: inventário shadcn<br/>+ custom variants + roll-your-own"]
    TokensFull --> CompFull["design-components<br/>specs primitivos completos"]
    CompScreen --> DTasks[design-tasks]
    CompFull --> DTasks
    DTasks --> Build[build]
    Build --> DReview["design-review<br/>screenshots via Playwright MCP"]
```

Path B (sem DESIGN_BRIEF.md do pm-handoff): roda `design-grill` antes de `design-brief-intake`.

---

## Backend flow

```mermaid
flowchart LR
    Flow[backend-flow] --> Brief[backend-brief-intake]
    Brief --> Stack[backend-stack]
    Stack --> Data[backend-data]
    Data --> API[backend-api]
    API --> Tasks[backend-tasks]
    Tasks --> Build[build]
    Build --> Review[backend-review]
```

---

## Frontend flow (com conflict check + designMode branching)

```mermaid
flowchart TD
    Flow[frontend-flow] --> FBrief[frontend-brief-intake]
    FBrief --> FStack["frontend-stack<br/>lê PROJECT_PROFILE.md<br/>aborta se designMode=shadcn-theme<br/>mas Component library ≠ shadcn/ui"]
    FStack --> FRoute[frontend-routing]
    FRoute --> FMode{designMode?}
    FMode -->|shadcn-theme| FCompShadcn["frontend-components<br/>npx shadcn add + custom variants<br/>+ roll-your-own plans"]
    FMode -->|custom-system| FCompFull["frontend-components<br/>specs por primitivo"]
    FCompShadcn --> FTasks[frontend-tasks]
    FCompFull --> FTasks
    FTasks --> Build[build]
    Build --> FReview["frontend-review<br/>screenshots via Playwright MCP"]
```

---

## Kanban flow

```mermaid
flowchart LR
    Flow[kanban-flow] --> Setup{Modo}
    Setup -->|setup 1ª vez| Preflight["pre-flight checks<br/>Atlassian MCP + projectKey<br/>+ issue types + Effort field"]
    Preflight --> Config[(".pm/jira-config.md")]
    Config --> Sync1["kanban-sync<br/>TASKS.md → Jira (idempotente)"]
    Setup -->|cycle| Sync2[kanban-sync]
    Setup -->|cycle| Status["kanban-status<br/>WIP + blockers + cycle time"]
    Setup -->|cycle| Pickup["kanban-pickup<br/>transition card + load contexto"]
    Sync1 -.JIRA_MAP.md.-> Sync2
```

---

## PROJECT_PROFILE.md — read/write map

```mermaid
flowchart LR
    PG[pm-grill] -->|escreve| PP[("PROJECT_PROFILE.md<br/>designMode<br/>uiFramework")]
    PI[pm-intake] -.detecta gap se<br/>UI framework ausente.-> PG
    PP -->|lê| PH[pm-handoff]
    PP -->|lê| DF[design-flow]
    PP -->|lê| DT[design-tokens]
    PP -->|lê| DC[design-components]
    PP -->|lê| FS[frontend-stack]
    PP -->|lê| FC[frontend-components]
    PH -.fallback prompt.-> PP
    DF -.fallback prompt.-> PP
    FS -.fallback prompt.-> PP
```

Projetos legados sem o arquivo: qualquer um dos três pontos (`pm-handoff`, `design-flow`, `frontend-stack`) pergunta uma vez e grava — sem re-rodar `pm-grill`.

---

## Artefatos por disciplina

| Disciplina | Pasta | Arquivos |
|---|---|---|
| PM | `.pm/<feature>/` | `INTAKE.md`, `GRILL_SUMMARY.md`, **`PROJECT_PROFILE.md`**, `PRD.md`, `ARCHITECTURE.md`, `WORKSTREAMS.md`, `TASKS.md`, `REVIEW.md`, `JIRA_MAP.md` |
| Design | `.design/<feature>/` | `DESIGN_GRILL.md` (Path B), `DESIGN_BRIEF.md`, `CODEBASE_AUDIT.md`, `IA.md`, `TOKENS.md` (slim ou full conforme modo), `COMPONENT_SPECS.md` (screen-centric ou full conforme modo), `DESIGN_TASKS.md`, `DESIGN_REVIEW.md`, `screenshots/` |
| Backend | `.backend/<feature>/` | `BACKEND_BRIEF.md`, `BACKEND_INTAKE.md`, `BACKEND_STACK.md`, `BACKEND_DATA.md`, `BACKEND_API.md`, `BACKEND_TASKS.md`, `BACKEND_REVIEW.md` |
| Frontend | `.frontend/<feature>/` | `FRONTEND_BRIEF.md`, `FRONTEND_INTAKE.md`, `FRONTEND_STACK.md`, `FRONTEND_ROUTES.md`, `COMPONENT_PLAN.md`, `FRONTEND_TASKS.md`, `FRONTEND_REVIEW.md` |
| Kanban | `.pm/` | `jira-config.md` (global, first-run) |

Todos são markdown plano — editáveis à mão, versionáveis, diffáveis. Re-rodar uma skill atualiza seu arquivo; artefatos downstream se adaptam na próxima execução.
