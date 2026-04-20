# Skills flow

All 43 skills, their artifacts, and the cross-cutting `PROJECT_PROFILE.md` contract rendered as mermaid diagrams. For a terminal-friendly ASCII view, see the main `README.md`.

---

## Top-level orchestration

```mermaid
flowchart TD
    Start([Idea or PRD draft]) --> PM[pm-flow]
    PM -->|pm-handoff| Design[design-flow]
    PM -->|pm-handoff| Backend[backend-flow]
    PM -->|pm-handoff| Frontend[frontend-flow]
    PM -->|pm-handoff| QA[qa-flow]
    Design -.specs upstream.-> Frontend
    Backend -.API contracts.-> Frontend
    Design --> Build1[implementação]
    Backend --> Build2[implementação]
    Frontend --> Build3[implementação]
    QA --> Build4[implementação]
    Build1 --> DReview[design-review]
    Build2 --> BReview[backend-review]
    Build3 --> FReview[frontend-review]
    Build4 --> QReview[qa-review]
    PM -.TASKS.md.-> Kanban[kanban-flow]
    Design -.DESIGN_TASKS.md.-> Kanban
    Backend -.BACKEND_TASKS.md.-> Kanban
    Frontend -.FRONTEND_TASKS.md.-> Kanban
    QA -.QA_TASKS.md.-> Kanban
    PM -->|pm-handoff| CC[cc-flow]
    CC --> Agents[".claude/agents/&lt;feature&gt;/"]
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
    Grill -.escreve.-> PP[("PROJECT_PROFILE.md<br/>designMode + uiFramework + testingRigor")]
    Handoff -.lê.-> PP
    Handoff --> DesignFlow[design-flow]
    Handoff --> BackendFlow[backend-flow]
    Handoff --> FrontendFlow[frontend-flow]
    Handoff --> QAFlow[qa-flow]
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

## QA flow (com testingRigor branching)

```mermaid
flowchart TD
    Flow["qa-flow<br/>lê PROJECT_PROFILE.md<br/>(prompt migração se faltar)"] --> Brief[qa-brief-intake]
    Brief --> Strategy[qa-strategy]
    Strategy --> Mode{testingRigor?}
    Mode -->|mvp| StratSlim["qa-strategy<br/>smoke pyramid<br/>unit crítico + 1 E2E por user journey<br/>apenas FR-XXX must"]
    Mode -->|full| StratFull["qa-strategy<br/>full pyramid<br/>coverage targets (80% lines / 70% branches)<br/>todos FR-XXX"]
    StratSlim --> Cases[qa-cases]
    StratFull --> Cases
    Cases --> QTasks[qa-tasks]
    QTasks --> Build[build]
    Build --> Review[qa-review]
    Review --> RMode{testingRigor?}
    RMode -->|mvp| RSlim["qa-review<br/>smoke report<br/>(sem visual regression / a11y matrix / perf budget)"]
    RMode -->|full| RFull["qa-review<br/>matrix + coverage + visual regression<br/>+ a11y matrix + perf budget"]
```

---

## CC flow

```mermaid
flowchart TD
    Flow[cc-flow] --> Setup{Modo}
    Setup -->|setup 1ª vez| Preflight["pre-flight checks<br/>confirma .pm/&lt;feature&gt;/PRD.md<br/>+ PROJECT_PROFILE.md"]
    Preflight --> Copy["copia templates/<br/>→ .cc-templates/"]
    Copy --> Config[(".pm/cc-config.md<br/>templatesVersion + overwritePolicy")]
    Config --> Dispatch{O que fazer?}
    Setup -->|cycle| Dispatch
    Dispatch -->|Sync| Sync[cc-sync]
    Dispatch -->|Preview| Preview["cc-sync --preview<br/>(dry-run)"]
    Dispatch -->|Setup only| Done([pronto])
    Sync --> Manifest[(".claude/.cc-manifest.md<br/>Owned + Archived + User-modified")]
    Sync --> Agents[(".claude/agents/&lt;feature&gt;/<br/>backend-engineer.md<br/>frontend-engineer.md<br/>qa-engineer.md<br/>product-architect.md<br/>design-reviewer.md")]
```

cc-sync lê `PROJECT_PROFILE.md` para selecionar partials: `designMode` escolhe entre `shadcn-theme-notes` ou `custom-system-notes`; `testingRigor` escolhe entre `testing-rigor-mvp` ou `testing-rigor-full`.

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
    PG[pm-grill] -->|escreve| PP[("PROJECT_PROFILE.md<br/>designMode<br/>uiFramework<br/>testingRigor")]
    PI[pm-intake] -.detecta gap se<br/>UI framework ausente.-> PG
    PP -->|lê| PH[pm-handoff]
    PP -->|lê| PA[pm-architecture]
    PP -->|lê| DF[design-flow]
    PP -->|lê| DT[design-tokens]
    PP -->|lê| DC[design-components]
    PP -->|lê| FS[frontend-stack]
    PP -->|lê| FC[frontend-components]
    PP -->|lê| QS[qa-strategy]
    PP -->|lê| QR[qa-review]
    PP -->|lê| CS[cc-sync]
    PH -.fallback prompt.-> PP
    DF -.fallback prompt.-> PP
    FS -.fallback prompt.-> PP
```

Projetos legados sem o arquivo: os três pontos de fallback-prompt (`pm-handoff`, `design-flow`, `frontend-stack`) perguntam uma vez e gravam — sem re-rodar `pm-grill`. Leitores downstream (`pm-architecture`, `design-tokens`, `design-components`, `frontend-components`, `qa-strategy`, `qa-review`) apenas consomem os valores já gravados.

---

## Artefatos por disciplina

| Disciplina | Pasta | Arquivos |
|---|---|---|
| PM | `.pm/<feature>/` | `INTAKE.md`, `GRILL_SUMMARY.md`, **`PROJECT_PROFILE.md`**, `PRD.md`, `ARCHITECTURE.md`, `WORKSTREAMS.md`, `TASKS.md`, `REVIEW.md`, `JIRA_MAP.md` |
| Design | `.design/<feature>/` | `DESIGN_GRILL.md` (Path B), `DESIGN_BRIEF.md`, `CODEBASE_AUDIT.md`, `IA.md`, `TOKENS.md` (slim ou full conforme modo), `COMPONENT_SPECS.md` (screen-centric ou full conforme modo), `DESIGN_TASKS.md`, `DESIGN_REVIEW.md`, `screenshots/` |
| Backend | `.backend/<feature>/` | `BACKEND_BRIEF.md`, `BACKEND_INTAKE.md`, `BACKEND_STACK.md`, `BACKEND_DATA.md`, `BACKEND_API.md`, `BACKEND_TASKS.md`, `BACKEND_REVIEW.md` |
| Frontend | `.frontend/<feature>/` | `FRONTEND_BRIEF.md`, `FRONTEND_INTAKE.md`, `FRONTEND_STACK.md`, `FRONTEND_ROUTES.md`, `COMPONENT_PLAN.md`, `FRONTEND_TASKS.md`, `FRONTEND_REVIEW.md` |
| QA | `.qa/<feature>/` | `QA_BRIEF.md`, `QA_INTAKE.md`, `QA_STRATEGY.md` (slim ou full conforme `testingRigor`), `TEST_CASES.md` (scoped por `testingRigor`), `QA_TASKS.md`, `QA_REVIEW.md` (slim ou full conforme `testingRigor`) |
| Kanban | `.pm/` | `jira-config.md` (global, first-run) |
| CC | `.cc-templates/` (consumer repo) | `VERSION`, `README.md`, `agents/*.md.tmpl`, `partials/*.md.tmpl` — copiado pelo cc-flow no primeiro run, consumer-editable depois |
| CC | `.claude/agents/<feature>/` (consumer repo) | `backend-engineer.md`, `frontend-engineer.md`, `qa-engineer.md`, `product-architect.md`, `design-reviewer.md` — gerados pelo cc-sync |
| CC | `.claude/` (consumer repo) | `.cc-manifest.md` (ownership + checksum ledger), `archive/<ts>/` (agentes de features removidas) |
| CC | `.pm/` | `cc-config.md` (global, first-run) |

Todos são markdown plano — editáveis à mão, versionáveis, diffáveis. Re-rodar uma skill atualiza seu arquivo; artefatos downstream se adaptam na próxima execução.
