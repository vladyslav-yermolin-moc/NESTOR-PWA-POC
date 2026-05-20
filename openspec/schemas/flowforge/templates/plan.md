## Overview

<!-- What needs to change and why. Reference the proposal and specs. -->

## Implementation Phases

<!-- Break the work into ordered phases. For each phase:
     - What to change (files, modules, components)
     - Acceptance criteria (how to verify the phase is done)
     - Dependencies on other phases
     - Whether it can run in parallel with other phases
     - Agent routing (which agents handle which tasks)
     - Human review checkpoint (what the reviewer should look for) -->

### Phase 1: [Phase title]

**Changes:**
<!-- What files/modules/components to modify or create -->

**Acceptance Criteria:**
<!-- How to verify this phase is complete -->

**Dependencies:** None | Phase N
**Parallel:** Yes/No — [reason if No]

**Agent Routing:**
<!-- Map task numbers to agent names from .claude/agents/.
     Example:
     - backend-dev: tasks 1.1, 1.2, 1.3 (PAR)
     - frontend-dev: tasks 1.4, 1.5 (PAR) -->

**Review Checkpoint:**
<!-- What the human reviewer should verify before proceeding -->

### Phase 2: [Phase title]

**Changes:**
<!-- ... -->

**Acceptance Criteria:**
<!-- ... -->

**Dependencies:** Phase 1
**Parallel:** Yes/No — [reason]

**Agent Routing:**
<!-- ... -->

**Review Checkpoint:**
<!-- ... -->

## Parallelization Notes

<!-- Summary of which phases can run concurrently for /opsx:implement.
     Example:
     - Wave 1 (parallel): Phase 1, Phase 2
     - Wave 2 (sequential, after Wave 1): Phase 3
     - Wave 3 (parallel): Phase 4, Phase 5 -->
