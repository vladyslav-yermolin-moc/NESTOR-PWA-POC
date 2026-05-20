# FlowForge Bootstrap Guide

Your project foundation is set up. Here's what to do next.

## Next Steps

1. [ ] Review foundational ADRs in `docs/decisions/` — edit anything that's wrong
2. [ ] Fill in any deferred ADR slots when ready (none deferred currently)
3. [ ] Review `external_docs` paths in `openspec/config.yaml` — add any docs created after bootstrap
4. [ ] Start your first feature: `/opsx:propose <feature-name>` (suggested: `pwa-shell` to wrap the prototype HTML)
5. [ ] Review specs + ADRs for first 2–3 features — check for cross-feature contradictions
6. [ ] Run `/opsx:apply` or `/opsx:implement` to build
7. [ ] Run `/opsx:archive` — establishes canonical spec baseline

## Key Commands

| Command | When to use |
|---|---|
| `/opsx:explore` | Think through ideas before committing |
| `/opsx:propose <name>` | Create change and generate all artifacts |
| `/opsx:apply [name]` | Implement tasks sequentially |
| `/opsx:implement [name]` | Implement with Agent Team (parallel) |
| `/opsx:archive [name]` | Archive completed change |

This file is disposable. Delete it once you've completed these steps.
