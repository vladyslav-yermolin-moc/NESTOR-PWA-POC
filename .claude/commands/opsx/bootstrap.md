---
description: >
  Two-phase project bootstrap for FlowForge. Phase 1 (Assess) detects
  project state, gathers foundational decisions through interactive
  consultation, and writes an assessment file for team review. Phase 2
  (Scaffold) reads the assessment and generates config.yaml, foundational
  ADRs, project brief, and a tailored bootstrap guide. Recommended but
  not required — experienced users can set up manually.
---

# /opsx:bootstrap — Project Bootstrap

Bootstrap a new or existing project for FlowForge. This command runs in two phases
with a human review gate between them.

## Invocation

```
/opsx:bootstrap
```

Re-run the same command after reviewing the assessment to trigger Phase 2.

## Phase Detection

On invocation, check which phase to run:

- If `openspec/bootstrap-assessment.md` does NOT exist → run **Phase 1 (Assess)**
- If `openspec/bootstrap-assessment.md` EXISTS → run **Phase 2 (Scaffold)**

---

## Phase 1 — Assess

### Step 1: Detect project state

Scan the working directory for signals:

| Check | How |
|---|---|
| Codebase exists? | Look for source directories (src/, app/, lib/, pkg/), language manifests (package.json, go.mod, Cargo.toml, requirements.txt, pom.xml, build.gradle), or significant source files |
| OpenSpec initialized? | Check for `openspec/` directory |

Classify based on findings:

| Signal | Classification |
|---|---|
| No codebase, no `openspec/` | **New project** |
| Codebase exists, no `openspec/` | **Existing project** |
| `openspec/` exists | **Already initialized** — warn: "This project already has OpenSpec configured. Re-running bootstrap will overwrite the assessment. Continue?" |

Present classification to the user and ask them to confirm or correct:

```
I detected: [classification]
[For existing projects: "I see a [language/framework] codebase with [key characteristics]."]

Is this a:
  (a) New project — no code yet
  (b) Existing project, in development (not released)
  (c) Existing project, released / in production

Your choice:
```

The user's answer is authoritative. Store as PROJECT_STATE.

### Step 2: Project brief gate

This is a hard gate. The bootstrap does not proceed without a project brief or equivalent.

Ask: "Do you have a project brief, PRD, or requirements document?"

- If YES → ask for the path. Read it and store as BRIEF_CONTENT. Continue.
- If NO (new project) → stop. Print:
  ```
  A project brief is required before bootstrapping. It defines the problem,
  target users, scope, and constraints that inform all foundational decisions.

  Create one using the template at:
    openspec/schemas/flowforge/templates/project-brief.md

  Save it to docs/project-brief.md, then re-run /opsx:bootstrap.
  ```
- If NO (existing project) → ask: "Can you describe the project in a few sentences?
  I'll draft a brief for you to review." Generate a brief from their description,
  save to `docs/project-brief.md`, and ask the user to review it before continuing.

### Step 3: Discover external documentation

Ask: "Do you have any existing documentation that should inform the SDLC workflow?
This includes PRDs, API references, research notes, architecture overviews,
design mockups, integration guides, or any other docs outside the codebase."

**For new projects:**

- If YES → ask: "List the paths (directories or files, relative to project root)."
  Store as EXTERNAL_DOCS (array of paths).
- If NO → set EXTERNAL_DOCS to `["docs/"]` (the default).

**For existing projects** — scan first, then confirm:

1. Look for common documentation locations:
   - `docs/`, `documentation/`, `wiki/`, `design/`, `specs/`, `api/`
   - Standalone files: `*.md` in root (excluding README, CHANGELOG, LICENSE, CONTRIBUTING)
   - `openapi.yaml`, `openapi.json`, `swagger.yaml`, `swagger.json`
   - `docs/adr/`, `docs/decisions/`, `docs/architecture/`

2. Present findings:
   ```
   I found documentation at:
     - docs/           (12 files — architecture notes, API reference)
     - openapi.yaml    (API spec)
     - design/         (4 files — mockups and wireframes)

   These will be read as context when generating proposals, designs, and specs.
   Add, remove, or confirm these paths:
   ```

3. User confirms, adds, or removes paths. Store as EXTERNAL_DOCS.

For each path, verify it exists. Warn (but don't block) if a path doesn't exist yet —
the user may plan to create it before starting feature work.

### Step 4: Gather foundational decisions

The six standard ADR slots. Each must be answered or explicitly deferred.

**For new projects** — ask each question sequentially:

1. **Tech stack (ADR-001):**
   "What language, framework, and runtime will you use?"
   Example: "TypeScript 5.x, React 19 + Next.js 15, Node.js 22"

2. **Architecture pattern (ADR-002):**
   "What architecture pattern?"
   Options: monolith, modular monolith, microservices, serverless, or describe your own.

3. **Data storage (ADR-003):**
   "What database and data access approach?"
   Example: "PostgreSQL 16 + Drizzle ORM" or "MongoDB + Mongoose"

4. **API style + auth (ADR-004):**
   "What API style and authentication approach?"
   Example: "REST with JWT auth" or "tRPC with session-based auth (Lucia)"

5. **Deployment model (ADR-005):**
   "What deployment model and CI/CD? Or defer to later?"
   Example: "Vercel + GitHub Actions" or "Deferred — will decide at first deploy"

6. **Testing strategy (ADR-006):**
   "What testing approach? Or defer to later?"
   Example: "Vitest unit + Playwright e2e, >80% coverage" or "Deferred — will decide at first feature"

**For existing projects** — scan the codebase first, then confirm:

1. **Tech stack (ADR-001):** Read language manifests (package.json, go.mod, etc.),
   detect frameworks from imports and config files. Present: "I found: [findings]. Correct?"

2. **Architecture pattern (ADR-002):** Analyze directory structure, service boundaries,
   module organization. Present: "This looks like a [pattern]. Correct?"

3. **Data storage (ADR-003):** Look for ORM imports, migration directories, DB config
   files, connection strings. Present findings.

4. **API style + auth (ADR-004):** Look for route definitions, GraphQL schemas, RPC
   definitions, auth middleware, session/token handling. Present findings.

5. **Deployment model (ADR-005):** Look for CI/CD configs (.github/workflows/,
   .gitlab-ci.yml, Jenkinsfile), Dockerfiles, infra-as-code. Present findings or
   note absence.

6. **Testing strategy (ADR-006):** Look for test frameworks in config/dependencies,
   test directories, coverage config. Present findings or note absence.

For each slot, the user can:
- **Confirm** the finding or answer
- **Correct** it with different information
- **Defer** it with a rationale and trigger condition

Store all answers as DECISIONS (array of 6 items, each with: slot, title, status [accepted|deferred], content, constraints, deferred_rationale, deferred_trigger).

### Step 5: Write the assessment file

Write `openspec/bootstrap-assessment.md` with all gathered information:

```markdown
# Bootstrap Assessment

- **Date:** [today's date]
- **Project state:** [new | existing-in-dev | existing-released]
- **Project brief:** [path to brief or "inline below"]

## Project Brief

[BRIEF_CONTENT — either path reference or inline content]

## External Documentation

Paths to include as read-only context for all artifact generation:

- [path 1]
- [path 2]
- ...

## Foundational Decisions

### ADR-001: Tech Stack
- **Status:** [accepted | deferred]
- **Decision:** [user's answer]
- **Constraints on future work:**
  - [derived constraints]
- **Deferred rationale:** [if deferred]
- **Deferred trigger:** [if deferred]

### ADR-002: Architecture Pattern
[same structure]

### ADR-003: Data Storage
[same structure]

### ADR-004: API Style and Auth
[same structure]

### ADR-005: Deployment Model
[same structure]

### ADR-006: Testing Strategy
[same structure]
```

Print:
```
Assessment saved to openspec/bootstrap-assessment.md.

Review this file with your team (PM + Solution Lead alignment recommended).
Edit anything that needs correction, then re-run /opsx:bootstrap to scaffold.
```

---

## Phase 2 — Scaffold

Triggered when `openspec/bootstrap-assessment.md` exists.

### Step 1: Read and validate the assessment

Read `openspec/bootstrap-assessment.md`. Validate:
- Project state is present
- Project brief is present or referenced
- External Documentation section is present (may be just `["docs/"]`)
- At least ADR-001 through ADR-004 have status `accepted` (005 and 006 may be deferred)

If validation fails, report what's missing and stop.

### Step 2: Run openspec init (if needed)

If the `openspec/` directory does not contain the standard OpenSpec structure
(no `specs/` or `changes/` subdirectories), run `openspec init`.

If `openspec/` already has the standard structure, skip this step.

### Step 3: Generate config.yaml

Write `openspec/config.yaml` using the assessment data:

```yaml
schema: flowforge

external_docs:
  - [from EXTERNAL_DOCS — one entry per discovered/confirmed path]

context: |
  # Project: [from brief]
  ## Tech Stack
  [from ADR-001 decision]
  ## Architecture
  [from ADR-002 decision]
  ## Data Layer
  [from ADR-003 decision]
  ## API & Auth
  [from ADR-004 decision]
  ## Deployment
  [from ADR-005 decision, or "Not yet decided — see ADR-005"]
  ## Testing
  [from ADR-006 decision, or "Not yet decided — see ADR-006"]
  ## Conventions
  [derive sensible defaults from tech stack — e.g., naming conventions,
   file organization patterns typical for the detected framework]

rules:
  proposal:
    - Include explicit "out of scope" section
    - Reference project brief for alignment
    - Identify all affected routes, services, and components explicitly
    - Note if a DB migration is required

  design:
    - Include component/service diagram
    - Sequence diagram required for any flow touching auth or multiple services
    - Document alternatives with rationale
    - Flag ADR-needed decisions
    - Reference foundational ADRs by number

  specs:
    - Use Given/When/Then format
    - RFC 2119 keywords (MUST, SHALL, SHOULD, MAY)
    - Organize by domain (specs/auth/, specs/billing/)
    - Include edge cases and error paths as separate scenarios
    - Each requirement gets at least two scenarios (happy path + failure)

  adrs:
    - Only create for substantive decisions
    - Use MADR format with Constraints on Future Work
    - Include Alternatives Considered
    - One decision per ADR
    - Status: Proposed → Accepted on archive

  plan:
    - Independently testable goals
    - SEQ or PAR tags per task
    - Agent routing section required — feeds /opsx:implement parallelism
    - Human review checkpoints
    - Rollback strategy for DB phases
    - Code samples per phase
    - Reference spec scenarios as acceptance criteria

  tasks:
    - Derived from plan.md — do not add tasks not in the plan
    - Group by phase, then by layer within each phase (backend / frontend / tests)
    - Each task references both its plan entry number and its spec scenario
    - Hierarchical numbering (1.1, 1.2)
    - Reference plan entry + spec scenario
```

Adapt the `rules:` section to include framework-specific conventions detected from
the tech stack. For example, SvelteKit projects should include server/client boundary
rules; Next.js projects should include server component rules.

### Step 4: Generate foundational ADRs

For each of the 6 ADR slots, generate a file in `docs/decisions/`:

- `docs/decisions/001-tech-stack.md`
- `docs/decisions/002-architecture-pattern.md`
- `docs/decisions/003-data-storage.md`
- `docs/decisions/004-api-style-and-auth.md`
- `docs/decisions/005-deployment-model.md`
- `docs/decisions/006-testing-strategy.md`

Use the `foundational-adr.md` template from `openspec/schemas/flowforge/templates/`. For each ADR:

- Set `status:` from the assessment (accepted or deferred)
- Set `category:` to the matching category value
- Set `date:` to today's date
- Fill "Context and Problem Statement" with project-relevant context from the brief
- Fill "Considered Options" from the assessment
  - For existing projects: include "Current implementation: [what codebase uses]" as the first option
- Fill "Decision Outcome" from the assessment
  - For deferred: "Deferred — [rationale]. Revisit when [trigger]."
- Fill "Constraints on Future Work" with actionable RFC 2119 constraints derived from the decision
- Fill "Deferred Aspects" if applicable

Create the `docs/decisions/` directory if it does not exist.

### Step 5: Handle project brief

- If the assessment references an existing brief by path → verify it exists. Done.
- If the assessment contains inline brief content → write it to `docs/project-brief.md`
  using the `project-brief.md` template structure.
- Create the `docs/` directory if it does not exist.

### Step 6: Generate bootstrap guide

Write `bootstrap-guide.md` at the project root.

**If PROJECT_STATE is "new":**

```markdown
# FlowForge Bootstrap Guide

Your project foundation is set up. Here's what to do next.

## Next Steps

1. [ ] Review foundational ADRs in docs/decisions/ — edit anything that's wrong
2. [ ] Fill in any deferred ADR slots when ready
3. [ ] Review external_docs paths in openspec/config.yaml — add any docs created after bootstrap
4. [ ] Start your first feature: /opsx:propose <feature-name>
5. [ ] Review specs + ADRs for first 2-3 features — check for cross-feature contradictions
6. [ ] Run /opsx:apply or /opsx:implement to build
7. [ ] Run /opsx:archive — establishes canonical spec baseline

## Key Commands

| Command | When to use |
|---|---|
| /opsx:explore | Think through ideas before committing |
| /opsx:propose <name> | Create change and generate all artifacts |
| /opsx:apply [name] | Implement tasks sequentially |
| /opsx:implement [name] | Implement with Agent Team (parallel) |
| /opsx:archive [name] | Archive completed change |

This file is disposable. Delete it once you've completed these steps.
```

**If PROJECT_STATE is "existing-in-dev" or "existing-released":**

```markdown
# FlowForge Bootstrap Guide

Your project foundation is set up. Foundational ADRs document your current
architecture. Here's what to do next.

## Next Steps

1. [ ] Review foundational ADRs in docs/decisions/ — these reflect codebase analysis, correct anything wrong
2. [ ] Fill in any deferred ADR slots when ready
3. [ ] Review external_docs paths in openspec/config.yaml — add any docs created after bootstrap
4. [ ] Start your next feature: /opsx:propose <feature-name>
5. [ ] Run /opsx:apply or /opsx:implement to build
6. [ ] Run /opsx:archive to finalize

## Key Commands

| Command | When to use |
|---|---|
| /opsx:explore | Think through ideas before committing |
| /opsx:propose <name> | Create change and generate all artifacts |
| /opsx:apply [name] | Implement tasks sequentially |
| /opsx:implement [name] | Implement with Agent Team (parallel) |
| /opsx:archive [name] | Archive completed change |

This file is disposable. Delete it once you've completed these steps.
```

### Step 7: Report

Print:
```
Bootstrap complete!

Generated:
  - openspec/config.yaml — project configuration (external_docs configured)
  - docs/decisions/001-006 — foundational ADRs
  - docs/project-brief.md — project brief
  - bootstrap-guide.md — your next steps

External docs configured: [list EXTERNAL_DOCS paths]
These will be read as context when generating proposals, designs, specs, and plans.

See bootstrap-guide.md for what to do next.
```
