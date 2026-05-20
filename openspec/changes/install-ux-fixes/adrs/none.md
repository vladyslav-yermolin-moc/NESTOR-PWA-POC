No architectural decisions required for this change.

All design choices for this change (safe-area padding strategy, `dvh` fallback, body-flex reset, inline install-banner block, UA-based platform detection, localStorage dismissal memory) are local design decisions within the envelope of existing ADRs (`docs/decisions/001-006` and `docs/decisions/2026-05-20-service-worker-caching-strategy.md`). They are documented in this change's `design.md` and do not warrant new ADR-level constraint records.
