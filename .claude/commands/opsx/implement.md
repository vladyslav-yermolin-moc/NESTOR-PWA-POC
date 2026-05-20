---
description: >
  Full parallel implementation pipeline for an OpenSpec change using Agent Teams.
  Reads plan.md for agent routing and parallelism — does not infer
  from task descriptions. Assembles a team from .claude/agents/, groups
  independent phases into parallel waves, runs spec-reader first, then
  executes waves with human review between each, then spec-verifier.
---

# /opsx:implement — Agent Team Pipeline

## Step 1 — Load spec context (blocking)

Invoke `spec-reader` synchronously. You need its output before proceeding.

Prompt: "Summarize change: $ARGUMENTS"

Store the full returned summary as SPEC_CONTEXT.

## Step 2 — Read the plan and compose the team

Read `openspec/changes/$ARGUMENTS/plan.md`.

Extract for each phase:
- Phase name and goal
- The agent routing section (which task numbers go to which agent role)
- PAR vs SEQ annotations per task
- Rollback strategy (store — needed if a phase fails)
- Dependencies on other phases

This is the authoritative source for what runs concurrently.
Do not re-derive routing from task descriptions.

Select 2–4 agents from `.claude/agents/` that best cover the phases
(see Agent Teams in CLAUDE.sdlc.md).

## Step 3 — Group phases into waves

Analyze phase dependencies from plan.md:
- Phases with no dependencies on each other are **independent** — group into the same wave
- Phases that depend on prior phases are **sequential** — place in a later wave

Present the team and wave strategy to the human:

```
Analyzing plan for $ARGUMENTS...

Team:
  backend-dev — Phase 1 (API layer), Phase 3 (data migration)
  frontend-dev — Phase 2 (UI components)
  tester — Phase 4 (integration tests)

Execution waves:
  Wave 1 (parallel): Phase 1 (backend-dev), Phase 2 (frontend-dev)
  Wave 2 (sequential, after Wave 1): Phase 3 (backend-dev)
  Wave 3 (parallel): Phase 4 (tester)

Each wave will be reviewed before the next begins.
Approve this strategy? (or suggest changes)
```

Wait for human approval. Adjust team composition or wave grouping if requested.

## Step 4 — Execute waves

For each wave in order:

### 4a — Spawn implementation teammates in parallel

For each phase in the wave, spawn one teammate per assigned agent using
`isolation: "worktree"`. Each teammate receives:
- Their agent role prompt from `.claude/agents/`
- The phase details and assigned tasks from plan.md
- SPEC_CONTEXT from spec-reader
- Relevant ADRs as constraints

Worktree strategy:
- Branch naming: `flowforge/$ARGUMENTS/phase-<N>/<agent-name>`
- All teammates in a wave spawn concurrently

Example (adapt agents based on team composition from Step 3):
```
Agent({
  agent: "<agent-name>",
  run_in_background: true,
  isolation: "worktree",
  prompt: `
Change ID: $ARGUMENTS
Phase: <phase name>
Tasks to implement: <task numbers and descriptions>
PAR/SEQ annotations: <from plan.md>
Spec context:
<SPEC_CONTEXT>
ADR constraints:
<relevant ADRs>
  `
})
```

### 4b — Collect results and merge

After all teammates in the wave complete:
- Merge worktree branches sequentially into the working branch
- Merge method: merge commit (preserves agent attribution)
- If two teammates modified the same file: stop, present conflict to human
- Worktrees are cleaned up after successful merge

Store list of all files created/modified across the wave.

### 4c — Spawn post-wave review teammates in parallel

Spawn `test-writer` and `code-reviewer` concurrently. Each receives:
- List of files modified in this wave
- SPEC_CONTEXT
- Change ID: $ARGUMENTS

### 4d — Handle review findings

Collect results from all post-wave teammates.

For any 🔴 Must Fix from code-reviewer:
- Re-invoke the relevant implementation teammate synchronously with the reviewer's feedback
- Re-run code-reviewer on the fixed files
- Repeat until no must-fix items remain

### 4e — Mark tasks complete

Check off completed tasks in `openspec/changes/$ARGUMENTS/tasks.md`.
Only mark a task done after its implementation passes code review and tests pass.

### 4f — Human review gate

```
Wave N complete:
  ✓ Phase X (agent): [summary of changes]
  ✓ Phase Y (agent): [summary of changes]
  Tests: N written, N passing
  Review: N must-fix (resolved), N should-fix, N suggestions

Merged to working branch. Approve to continue to Wave N+1?
```

Wait for approval before proceeding to the next wave.

### 4g — Wave rollback (if a phase fails)

If any teammate returns a fatal error:
- Read the rollback strategy for the failed phase from plan.md
- Report the rollback steps — do NOT execute DB rollbacks automatically
- Stop pipeline and wait for human decision

## Step 5 — Final verification (blocking)

After all waves complete, invoke `spec-verifier` synchronously:

Prompt: "Verify change: $ARGUMENTS"

If spec-verifier returns any ❌ FAIL or ⚠️ PARTIAL:
- Report them clearly
- Do NOT proceed to archive

## Step 6 — Report

```markdown
## Implementation Complete: <change-id>

### Waves
- Wave 1: Phase 1 (<agent>), Phase 2 (<agent>) — ✅ complete / ❌ failed
- Wave 2: Phase 3 (<agent>) — ✅ complete / ❌ failed

### Files modified
<list all files touched across all teammates>

### Test summary
<N> tests written, <N> passing

### Spec compliance
<paste spec-verifier summary>

### ADR compliance
<spec-verifier ADR results>

### Ready for archive?
YES — run /opsx:archive <change-id>
NO  — <list gaps>
```
