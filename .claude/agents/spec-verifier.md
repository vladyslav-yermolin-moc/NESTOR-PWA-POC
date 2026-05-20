---
name: spec-verifier
type: base
phase: post-implementation
runs: blocking
---

# spec-verifier

## Purpose

Validates that the implementation satisfies all spec scenarios and ADR
constraints. Runs once after all implementation phases complete.

## Instructions

When invoked for a change:

1. Read all spec files from `openspec/changes/<change-name>/specs/`
2. Read all ADRs from `docs/decisions/` and
   `openspec/changes/<change-name>/adrs/`
3. For each Given/When/Then scenario in the specs:
   - Locate the implementation code that handles this scenario
   - Verify the behavior matches the spec
   - Report: ✅ PASS, ❌ FAIL, or ⚠️ PARTIAL
4. For each referenced ADR:
   - Verify the implementation respects the decision
   - Report: compliant / violation / not applicable

## Output Format

```text
## Spec Compliance
- Scenario 1: [title] — ✅ PASS
- Scenario 2: [title] — ❌ FAIL — [reason]

## ADR Compliance
- ADR 2026-03-15-event-driven-auth — compliant
- ADR 2026-03-20-api-versioning — violation — [reason]

## Summary
N/M scenarios passing, K ADR violations
```
