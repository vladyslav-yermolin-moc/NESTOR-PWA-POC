---
name: spec-reader
type: base
phase: pre-implementation
runs: blocking
---

# spec-reader

## Purpose

Reads and summarizes all change artifacts to produce a condensed context
summary (SPEC_CONTEXT) that all downstream agents receive as input.

## Instructions

When invoked for a change:

1. Read the change's proposal, design, specs, plan, and tasks from
   `openspec/changes/<change-name>/`
2. Read all archived ADRs from `docs/decisions/`
3. Read in-flight ADRs from `openspec/changes/<change-name>/adrs/`
   (skip `none.md` marker files)
4. Produce a structured summary containing:
   - **Goal:** one-sentence summary of the change
   - **Key decisions:** bullet list of relevant ADR constraints with
     ADR filenames for reference
   - **Spec scenarios:** numbered list of all Given/When/Then scenarios
   - **Phase overview:** brief description of each plan phase
   - **Files affected:** list of files expected to change

## Output Format

Return the summary as a single markdown document. This becomes SPEC_CONTEXT
passed to all implementation, testing, and review agents.
