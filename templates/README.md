# cc-templates starter kit

This directory ships with project-skills and contains starter templates for `cc-sync`. On the first run of `cc-flow` in a consumer project, these files are copied into `.cc-templates/` where they become consumer-editable.

**Once copied to `.cc-templates/`, these source files are not used again.** All subsequent `cc-sync` runs read from `.cc-templates/` in the consumer project.

---

## Directory structure

```
templates/
  VERSION                              # semver — bump on breaking template changes
  README.md                            # this file
  agents/
    backend-engineer.md.tmpl           # backend implementation agent
    frontend-engineer.md.tmpl          # frontend implementation agent
    qa-engineer.md.tmpl                # QA/test agent
    product-architect.md.tmpl          # scope + requirements advisor
    design-reviewer.md.tmpl            # design spec compliance reviewer
  partials/
    testing-rigor-mvp.md.tmpl          # included in qa-engineer for mvp projects
    testing-rigor-full.md.tmpl         # included in qa-engineer for full projects
    shadcn-theme-notes.md.tmpl         # included in frontend-engineer for shadcn-theme projects
    custom-system-notes.md.tmpl        # included in frontend-engineer for custom-system projects
```

---

## Editing conventions

After `cc-flow` copies these into `.cc-templates/`, they are yours to customize. Common edits:

- **Tone and style**: adjust how agents introduce themselves, how terse or verbose they are.
- **Company conventions**: add your coding standards, naming conventions, PR workflow, etc. to the Conventions section of each agent.
- **Do-not rules**: add project-specific guardrails (e.g., "never use `any` in TypeScript", "always use transactions for multi-table writes").
- **Allowed tools**: adjust the `allowed-tools` frontmatter to match what your team has installed (Playwright, Puppeteer, specific MCP servers).

All `{{var}}` placeholders will still be substituted by cc-sync on each run. Your custom prose around them is preserved.

---

## VERSION semantics

`VERSION` contains a semver string (e.g., `1.0.0`). cc-sync compares `.cc-templates/VERSION` to the project-skills starter `templates/VERSION` on each run:

- **Match**: no warning.
- **Mismatch**: cc-sync warns "starter is v`<new>` but your `.cc-templates/` is v`<old>`". You decide whether to diff and pull changes.

**When VERSION is bumped in project-skills:**
- **Patch** (1.0.0 → 1.0.1): bug fix in template prose — safe to ignore or adopt.
- **Minor** (1.0.0 → 1.1.0): new variable or new partial — your existing agents degrade gracefully (missing vars render as empty string); adopt when ready.
- **Major** (1.0.0 → 2.0.0): breaking structural change — review the diff before adopting.

cc-sync never auto-overwrites `.cc-templates/`. You always control when to adopt upstream changes.

---

## How partials work

Partials are short `.md.tmpl` files that cc-sync chooses and inserts based on `PROJECT_PROFILE.md`. Templates reference them with `{{> partial-name}}` but contain **no conditional logic themselves**.

cc-sync makes the choice:

| Condition | Partial included in |
|---|---|
| `designMode=shadcn-theme` | `shadcn-theme-notes.md.tmpl` → `frontend-engineer.md.tmpl` |
| `designMode=custom-system` | `custom-system-notes.md.tmpl` → `frontend-engineer.md.tmpl` |
| `testingRigor=mvp` | `testing-rigor-mvp.md.tmpl` → `qa-engineer.md.tmpl` |
| `testingRigor=full` | `testing-rigor-full.md.tmpl` → `qa-engineer.md.tmpl` |

This keeps each template file short and scannable — partials decompose the branching.

---

## How to add a new agent template

1. Create `agents/<your-agent>.md.tmpl` with this frontmatter:

```
---
name: {{feature}}-<your-agent>
description: <one-line description with {{feature}} interpolated>
allowed-tools: Read, Edit, Write, Bash, Grep, Glob
---
```

2. Body structure (see existing agents for reference):
   - `## Who you are` — role + feature + what this agent owns
   - `## Stack` — relevant stack fields from `{{stack.*}}` vars
   - `## Conventions` — `{{conventions.<discipline>}}` or custom rules
   - `## Key artifacts` — `{{artifactPaths.*}}` paths for the agent to reference
   - `## Do not` — negative guardrails
   - `## Handoff boundaries` — when to defer to another agent

3. cc-sync will pick up the new template on the next run and add it to `.claude/agents/<feature>/`.

---

## How to add a new partial

1. Create `partials/<name>.md.tmpl`.
2. In cc-sync, add the selection logic: read the appropriate PROJECT_PROFILE.md field and choose which partial to insert.
3. Reference it in the target agent template with `{{> <name>}}`.

If you want a partial that's always included (no condition), add `{{> <name>}}` to the template and ensure cc-sync always resolves it.

---

## Forking for company-wide use

To maintain a company-wide base set of templates:

1. Fork this repo (or just copy the `templates/` directory into your own internal repo).
2. Customize to your company's stack, conventions, and agent styles.
3. Update `VERSION` with your own semver.
4. Distribute to teams: point them to your fork's `templates/` directory and instruct them to copy manually into `.cc-templates/` (same pattern as the project-skills first-run copy).

Remote template fetching (auto-pulling from a URL at sync time) is out of scope for v1. The bundled-starter + consumer-editable model covers the common case.

---

## Relationship to docs/guardrails.md

`docs/guardrails.md` is the top-level consumer explainer for the entire cc-flow family — what it is, why it exists, the full end-to-end flow, the manifest, the archive lifecycle, and FAQ. This file (`templates/README.md`) is narrowly scoped to the starter kit itself. Read `docs/guardrails.md` for the big picture.
