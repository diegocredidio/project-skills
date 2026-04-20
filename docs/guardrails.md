# Claude Code Guardrails — cc-flow

The planning artifacts produced by project-skills (`.pm/`, `.design/`, `.backend/`, `.frontend/`, `.qa/`) capture every project decision — stack, data model, API contracts, tokens, component specs, test strategy. But those artifacts are passive: when a developer opens Claude Code to implement a task, they must manually load the relevant context in each conversation.

`cc-flow` and `cc-sync` compile those artifacts into **executable guardrails** inside the developer's project — per-feature Claude Code agents in `.claude/agents/<feature>/` that come pre-loaded with stack knowledge, conventions, token names, component inventory, pyramid + tooling, coverage targets, and explicit do/don't rules. The developer invokes `@<feature>-backend-engineer` and starts building with full context already in place.

---

## What cc-flow generates

For each feature, `cc-sync` renders up to 5 agent files into `.claude/agents/<feature>/`:

| Agent | Role | Reads |
|---|---|---|
| `<feature>-backend-engineer.md` | Backend implementation | BACKEND_STACK, BACKEND_DATA, BACKEND_API, ARCHITECTURE |
| `<feature>-frontend-engineer.md` | Frontend implementation | FRONTEND_STACK, TOKENS, COMPONENT_SPECS, FRONTEND_ROUTES |
| `<feature>-qa-engineer.md` | Tests + QA tasks | QA_STRATEGY, TEST_CASES |
| `<feature>-product-architect.md` | Scope + requirements advisor (read-only) | PRD, ARCHITECTURE, WORKSTREAMS |
| `<feature>-design-reviewer.md` | Design spec compliance (read-only) | TOKENS, COMPONENT_SPECS |

Agents live at `.claude/agents/<feature>/` — one folder per feature, mirroring the artifact layout (`.backend/<feature>/`, `.design/<feature>/`, etc.).

---

## Why

Before cc-flow, the pipeline looked like this:

```
PRD → Architecture → Tasks → Jira (via kanban-sync) → dev picks card (kanban-pickup) → ???
```

The `???` was: the developer opens a new Claude Code conversation, manually pastes in the relevant stack doc, the API contracts, the token list, the test strategy. Inconsistent across devs, lost between sessions, and invisible to new team members joining mid-feature.

With cc-flow:

```
PRD → Architecture → Tasks → Jira → kanban-pickup → @<feature>-backend-engineer (pre-loaded)
```

Every task lands the same way, for every developer, every session. The spec compiles to context.

---

## End-to-end flow

1. **Run `pm-flow`** — produces `.pm/<feature>/PRD.md`, `ARCHITECTURE.md`, `WORKSTREAMS.md`, `TASKS.md`, and `PROJECT_PROFILE.md` (with `designMode` and `testingRigor`).
2. **Run discipline flows** — `design-flow`, `backend-flow`, `frontend-flow`, `qa-flow` produce their respective artifact trees.
3. **Run `pm-handoff`** — briefs each discipline and optionally invokes `cc-flow` as the 5th option.
4. **First run of `cc-flow` (setup mode)**:
   - Pre-flight: confirms planning artifacts exist.
   - Copies `templates/` from the project-skills install into `.cc-templates/` in the consumer project.
   - Writes `.pm/cc-config.md` with template version, scope, and overwrite policy.
5. **`cc-flow` cycle mode → `cc-sync`**:
   - Loads PROJECT_PROFILE, PRD, stacks, tokens, routes, QA strategy for the feature.
   - Extracts ~30 template variables from the artifacts.
   - Selects partials based on `designMode` and `testingRigor` (e.g., `shadcn-theme-notes` or `testing-rigor-full`).
   - Renders 5 agent templates with variables + partials substituted.
   - Shows a preview table (`[CREATE]`/`[UPDATE]`/`[PROMPT]`/`[SKIP]`) and waits for confirmation.
   - Writes files to `.claude/agents/<feature>/`, saves SHA-256 checksums to `.claude/.cc-manifest.md`.
6. **Developer runs `kanban-pickup`** — picks a task from the board; if `.claude/agents/<feature>/` exists, kanban-pickup suggests the matching agent.
7. **Developer invokes `@<feature>-backend-engineer`** (or frontend, qa, etc.) — Claude Code loads the agent with full context from the planning artifacts.

---

## Per-feature scoping

Agents are scoped per-feature, not per-project. Each feature gets its own folder: `.claude/agents/user-auth/`, `.claude/agents/dashboard/`, etc.

**Why:** a project with multiple active features in parallel would have `backend-engineer.md` flip its definition every time a different feature's artifacts were last synced. Per-feature scoping means 10 features → 10 independent agent sets, no conflicts, clean archive lifecycle when a feature ships.

**Tradeoff:** more files (5 agents × N features). Mitigated by the subfolder layout and the manifest index.

---

## The starter template kit

The `templates/` directory in this repo ships a starter set of 5 agent templates and 4 partials. On the first run of `cc-flow`, these are copied to `.cc-templates/` in the consumer project.

**After the copy, `.cc-templates/` is yours.** Edit it to match your company's stack defaults, tone, conventions, and guardrails. cc-sync reads from `.cc-templates/` — never from the project-skills source — on all subsequent runs.

**`VERSION` semantics**: `.cc-templates/VERSION` is compared to project-skills' `templates/VERSION` on every sync run. A mismatch surfaces a warning; it does not trigger auto-overwrite. Patch bumps are safe to ignore; minor bumps add new variables or partials; major bumps are breaking — review the diff before adopting.

**Forking for company-wide use**: copy `templates/` into an internal repo, customize, maintain your own `VERSION`. Distribute to teams as a one-time copy into their `.cc-templates/`. Remote template fetching (pull-on-sync from a URL) is out of scope for v1.

See [`templates/README.md`](../templates/README.md) for detailed editing conventions, how to add new agent templates or partials, and the VERSION update workflow.

---

## Editing `.cc-templates/`

After setup, `.cc-templates/` is a regular directory in your repo. Common edits:

- **Agent body**: adjust tone, add company conventions, extend the "Do not" list.
- **Partials**: customize `testing-rigor-full.md.tmpl` to your actual coverage thresholds, or `shadcn-theme-notes.md.tmpl` to your shadcn version and variant conventions.
- **New agents**: add a `<name>.md.tmpl` — cc-sync picks it up on the next run.
- **New partials**: add to `partials/`, then add selection logic in cc-sync's Step 3.

Templates use `{{var}}` for variable interpolation and `{{> partial-name}}` for partial includes. No `{{#if}}` or `{{#each}}` — branching lives in cc-sync, not in templates.

---

## Checksum-prompt policy

cc-sync tracks the SHA-256 of every file it writes in `.claude/.cc-manifest.md`. On re-sync:

- **SHA matches manifest** → file unmodified since last sync → overwrite silently with new rendered content.
- **SHA differs from manifest** → developer edited the file → cc-sync prompts: "Keep your edits (k) or overwrite with re-rendered template (o)?"
- **File missing from disk** → developer deleted it → cc-sync skips and logs; it does not regenerate uninvited.

User-edited files are tracked in a separate "User-modified" section of the manifest so the divergence is visible on every subsequent `--preview` run.

---

## The manifest

Path: `.claude/.cc-manifest.md`. Three sections:

**Owned** — files cc-sync manages: path, feature, source template, SHA, write timestamp.

**Archived** — files whose source feature was removed upstream: original path, archive path, timestamp.

**User-modified** — files where the developer's SHA diverged: path, manifest SHA, current SHA, detected-at.

Invariants:
- cc-sync never touches files absent from the manifest. Ownership is explicit.
- Manifest is saved incrementally (after each file write) — partial writes survive mid-run failures.
- Re-runs read the manifest to classify each file before writing anything.

---

## Archive lifecycle

When a feature's artifact folders (`.pm/<feature>/`, `.backend/<feature>/`, etc.) are removed, cc-sync detects the orphaned agents on the next sync and:

1. Moves each owned agent file to `.claude/archive/<ISO-timestamp>/<feature>/`.
2. Updates the manifest row from Owned → Archived.
3. Reports: "Archived N agents for feature `<feature>` (source removed upstream)."

Files are never deleted. `.claude/archive/` can be pruned manually when you're comfortable. This preserves history for audits and post-mortems.

---

## Integration with kanban-pickup

`kanban-pickup` closes the final hop. After a developer transitions a Jira card to "In Progress", kanban-pickup checks whether `.claude/agents/<feature>/` exists and — if it does — surfaces the matching agent:

> "Task picked: BACKEND_TASK_003. Invoke the feature's specialized agent: `@<feature>-backend-engineer` — it has the stack, conventions, and data model preloaded from the planning artifacts."

This makes the `spec → Jira → pickup → agent` chain automatic.

---

## Out of scope for v1

- **Remote template source**: pulling templates from a company-owned URL at sync time. Manual copy + consumer-editable covers the common case.
- **`cc-status` as a separate skill**: `cc-sync --preview` (dry-run mode) provides the same visibility.
- **`ccGuardrails` field in PROJECT_PROFILE.md**: presence of `.cc-templates/` is sufficient signal; no new schema field needed.
- **Project-level aggregate agents**: cross-feature `backend-engineer.md` for onboarding. Revisit when a team asks.
- **Git pre-commit hook for drift check**: document as manual practice; no automation in v1.
- **`cc-diff <feature>` tool**: diff between manifest SHA and current file. Nice-to-have for v1.x.

---

## FAQ

**Can I skip cc-flow?** Yes. cc-flow is opt-in. You can run `pm-flow`, `backend-flow`, etc. and use the artifact markdown files directly in your Claude Code conversations without ever running cc-flow.

**Can I regenerate just one agent?** Yes. Run `cc-sync` and select the feature — the preview will show `[UPDATE]` for existing agents. You can also delete a single agent file; cc-sync will classify it as `[SKIP-DELETED]` (missing from disk) and not regenerate unless you remove its manifest entry.

**Does cc-sync touch files it didn't write?** No. The manifest is the sole ownership ledger. Files not in the manifest's Owned section are invisible to cc-sync.

**What if I add a completely custom agent file to `.claude/agents/<feature>/`?** It's ignored by cc-sync (not in the manifest). You manage it manually.

**What happens when I update an artifact (e.g., add a new endpoint to BACKEND_API.md)?** Re-run `cc-sync` for the feature. It re-renders the `backend-engineer` agent with the updated `{{apiEndpoints}}` variable. If you haven't edited the agent since last sync, it overwrites silently. If you have edited it, it prompts.

**Can I use cc-flow without kanban?** Yes. cc-flow and kanban-flow are independent. cc-flow reads artifact markdown; it doesn't need Jira.
