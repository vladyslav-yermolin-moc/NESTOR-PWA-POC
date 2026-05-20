# OpenSpec SDLC Configuration — FlowForge

> Drop-in file. Requires the FlowForge schema installed at `openspec/schemas/flowforge/`.

## Readiness Check

**Before running any `/opsx:*` command that generates artifacts**, read `openspec/config.yaml` and check the `context:` section. If the context is empty, only contains comments (lines starting with `#`), or still has the default placeholder text (`# Fill in your project's tech stack`), **stop immediately** and tell the user:

> **FlowForge is not ready yet.** The `context:` section in `openspec/config.yaml` is empty or still has the placeholder text. Agents need your project's tech stack, conventions, and constraints to generate useful artifacts.
>
> Open `openspec/config.yaml` and fill in the `context:` section. Example:
> ```yaml
> context: |
>   Tech stack: Go, PostgreSQL, gRPC
>   Testing: go test, testcontainers
>   Style: golangci-lint, strict types
> ```
>
> Then re-run your command.

This check applies to: `/opsx:propose`, `/opsx:apply`, `/opsx:implement`. It does NOT block `/opsx:explore`, `/opsx:bootstrap`, or `/opsx:archive`.

## Project Bootstrap

For new project setup or onboarding an existing project to FlowForge, use `/opsx:bootstrap`. This is a two-phase command that establishes your project foundation (config.yaml, foundational ADRs, project brief) before you start creating features.

## Artifact Chain

```
proposal → design → specs → plan → tasks
                      │
                      └──► adrs (conditional)
```

1. **proposal** — Why the change is needed, scope, and impact.
2. **design** — Technical approach and architecture decisions.
3. **specs** — Delta specs (ADDED/MODIFIED/REMOVED) with Given/When/Then scenarios.
4. **adrs** — *Conditional.* Formalized architectural decisions (MADR format). Created when substantive decisions exist; otherwise `adrs/none.md`.
5. **plan** — Implementation blueprint with phases, agent routing, and parallelization notes.
6. **tasks** — Brief checkbox checklist derived from the plan.

## Commands

- `/opsx:bootstrap` — set up project foundation (config, ADRs, project brief)
- `/opsx:explore` — explore ideas before committing to a change
- `/opsx:propose <name>` — create change and generate all artifacts with gates
- `/opsx:apply [name]` — implement tasks sequentially (single agent)
- `/opsx:implement [name]` — implement with parallel agents (see `.claude/commands/opsx/implement.md`)
- `/opsx:archive [name]` — archive the completed change

## Human Approval Gates

After generating each artifact, **stop** and present a summary. Then offer three options:

1. **Approve** — proceed to the next artifact in the chain
2. **Regenerate** — user provides feedback, regenerate the artifact incorporating it
3. **Edit manually** — user edits files directly, then re-run `/opsx:propose` to proceed

`/opsx:propose` generates artifacts one at a time, stopping at each gate for approval.

Gates are defined in `openspec/schemas/flowforge/schema.yaml` under the `gates:` key.

## ADR Enforcement

ADRs are first-class constraints at every stage.

### Before Generating Any Artifact

1. Read all existing ADRs from `docs/decisions/`
2. Read ADRs from `openspec/changes/<change-name>/adrs/` (if any)
3. Identify ADRs whose scope overlaps with the artifact being generated
4. Cite relevant ADRs explicitly (e.g., "Per ADR 2026-03-15-event-driven-auth: ...")

Requirements per artifact:
- **design** — must cite relevant ADRs as constraints
- **specs** — must align scenarios with existing ADR decisions
- **plan** — must incorporate ADR constraints into phase design and acceptance criteria

### During Implementation

- Agent context (SPEC_CONTEXT) includes ADR summaries
- Implementation agents receive ADRs as constraints — violations are bugs
- Code reviewer checks ADR compliance
- Spec verifier validates ADR compliance in final report

### ADR Conflicts

When an ADR conflicts with the current change:
1. Flag the conflict at the gate
2. Do not proceed until the user resolves (update/supersede the ADR or adjust the change)

## ADR Format and Lifecycle

- **During work:** `openspec/changes/<change-name>/adrs/YYYY-MM-DD-<title>.md`
- **After archive:** Copied to `docs/decisions/` (configurable via `adr_archive_path` in config)
- **Format:** MADR (see template)
- **Skip marker:** `adrs/none.md` with "No architectural decisions required for this change."
- **Standalone:** ADRs can be created directly at `docs/decisions/` outside any change

### When to Create ADRs

After creating specs, analyze design and specs. Create an ADR when:
- A technology/framework/library choice was made between alternatives
- A structural pattern was chosen (e.g., event-driven vs polling)
- A trade-off was explicitly evaluated (performance vs simplicity)
- An API contract or data model was established that constrains future work

## Archive Behavior

When running `/opsx:archive`:

1. Check for ADR files in `openspec/changes/<change-name>/adrs/`
2. Skip `none.md` marker files
3. Copy all substantive ADR files to the project-level ADR directory (default: `docs/decisions/`; override with `adr_archive_path` in `openspec/config.yaml`). Create the directory if it doesn't exist.
4. Promote each archived ADR's status from `proposed` to `accepted`
5. Proceed with standard OpenSpec archive (delta spec merge, move to archive folder)

**Note:** There is no separate human approval gate after ADR generation. ADRs are reviewed as part of the specs gate — they are conditional artifacts that branch off specs and are validated together before proceeding to the plan.

---

## Agent Teams — Dynamic Composition

The lead session can assemble a team from the agent pool in `.claude/agents/` for any task that benefits from parallel, coordinated work.

### How It Works

1. Analyze the task — understand scope, type, and expected deliverable
2. Select **2–4 agents** from the pool that best cover the task (3 is optimal)
3. Spawn them as teammates with task-specific context in their spawn prompts
4. Define a shared task list with dependencies
5. Let the team work — agents communicate directly, claim tasks, share findings

### Base Agents (shipped by FlowForge)

| Agent | Phase | Runs | Purpose |
|-------|-------|------|---------|
| `spec-reader` | pre-implementation | blocking | Summarizes change context for downstream agents |
| `spec-verifier` | post-implementation | blocking | Validates spec + ADR compliance |
| `test-writer` | post-phase | parallel | Writes tests for modified files |

### Role Agents (shipped by FlowForge)

| Agent | Role | Best for |
|-------|------|----------|
| `architect` | Solution/system architect | System design, ADRs, component boundaries |
| `backend-dev` | Backend developer | Server-side code, APIs, DB, integrations |
| `frontend-dev` | Frontend developer | UI, components, client-side logic |
| `mobile-dev` | Mobile developer | Mobile apps, native features |
| `devops` | DevOps/Platform engineer | CI/CD, infra, deployment, monitoring |
| `tester` | QA engineer | Tests, test plans, regression, verification |
| `skeptic` | Devil's advocate | Challenge assumptions, find risks, debate |
| `code-reviewer` | Code reviewer | Code quality, PR reviews, standards |
| `designer` | UX/UI designer | User flows, wireframes, interaction design |
| `business-analyst` | Business analyst | Requirements, stories, estimation, plans |
| `ai-engineer` | AI/ML engineer | LLM, chatbots, RAG, voice AI, prompts |
| `security-engineer` | Security engineer | Vulnerabilities, OWASP, auth audit |

### Composition Guidelines

**Implementation** (write code):
→ developers for relevant layers + `tester`

**Architecture & design** (decisions):
→ `architect` + `skeptic` + specialist

**Planning & estimation**:
→ `business-analyst` + relevant developer + `skeptic` or `architect`

**Code review / audit**:
→ `code-reviewer` + `security-engineer`

**Research & evaluation**:
→ domain specialist + `business-analyst`

**AI/chatbot/voice tasks**:
→ `ai-engineer` + `backend-dev` + `tester` or `architect`

These are guidelines, not rigid rules. Adapt based on the actual task.

### Integration with OpenSpec

Agent Teams complement the OpenSpec workflow:

- **During `proposal` / `design`**: use `architect` + `skeptic` + `business-analyst` to debate the approach before committing to artifacts
- **During `specs`**: use `business-analyst` + `tester` to write thorough specifications with edge cases
- **During `adrs`**: use `architect` + `skeptic` — the skeptic challenges, the architect defends, produces stronger ADRs
- **During `/opsx:implement`**: Agent Teams are the execution mechanism — each teammate owns a phase, they coordinate directly instead of through the lead
- **During verification**: use `tester` + `code-reviewer` + `security-engineer` for comprehensive validation

### Coordination Rules

- **Delegate mode ON** — the lead coordinates, does not implement
- Teammates message each other directly — no relaying through the lead
- Each teammate reads existing code and CLAUDE.md before starting
- Tasks have explicit dependencies — blocked tasks wait
- **3 agents is the sweet spot** — more only if the task genuinely needs it
- Before closing, verify all tasks complete, then synthesize results

---

## /opsx:implement — Agent Team Parallel Execution

`/opsx:implement` is an alternative to `/opsx:apply` that assembles an Agent Team for parallel, coordinated implementation. Both commands can be used on the same change interchangeably.

- Use `/opsx:apply` for simple changes or when sequential execution is fine
- Use `/opsx:implement` when the plan has independent phases that benefit from parallel execution by specialized agents

The command groups independent phases into **waves** that run in parallel, with human review between each wave. Base agents (`spec-reader`, `spec-verifier`) run as blocking steps before and after implementation. Post-phase agents (`test-writer`, `code-reviewer`) run after each wave to validate changes.

For the full execution protocol, see `.claude/commands/opsx/implement.md`.

---

## External Documentation

Documentation created outside the SDLC cycle (e.g. PRDs, research notes, API references, architecture overviews) can be made available to all OpenSpec commands by configuring `external_docs` in `openspec/config.yaml`:

```yaml
external_docs:
  - docs/
  - path/to/other/docs/
```

### Behavior

- Paths are relative to the project root.
- Each entry can be a directory (all files inside are included) or a specific file path.
- When creating any artifact (proposal, design, specs, adrs, plan, tasks), the agent reads the external docs for additional context before generating output.
- External docs are **read-only context** — the SDLC workflow never modifies them.
- If a directory doesn't exist, it is silently skipped.

### When to Use

- You have product requirements, design mockups, or research docs that inform the change but live outside `openspec/`.
- API documentation or integration guides that specs and designs should reference.
- Existing architecture docs or decision logs maintained separately from ADRs.

---

## Working with Multiple Changes

### Switching Changes

Specify the change name explicitly in any command:

```
/opsx:propose my-feature
/opsx:apply fix-auth-bug
/opsx:implement add-dark-mode
```

Without a name, commands operate on the most recently active change.

### Parallel Work

Multiple changes can exist simultaneously in `openspec/changes/`. Each is independent. Use `/opsx:implement` or `/opsx:apply` with explicit names to switch between them.
