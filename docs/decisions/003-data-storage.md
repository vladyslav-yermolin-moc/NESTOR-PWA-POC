---
status: accepted
date: 2026-05-19
category: data-storage
---

# ADR-003: Data Storage — None (Inline Fixtures Only)

## Context and Problem Statement

The POC demonstrates a PWA install experience and the look-and-feel of the buyer flow. It does not demonstrate live MLS search, persistent user data, or backend integration. The decision is how to source the listing data that the prototype displays.

## Decision Drivers

- **POC scope is install + UI feel.** Nora is testing whether PWA technology is the right delivery vehicle, not whether the data layer works.
- **The prototype already inlines its data.** `2026-05-19-mvp-showing.html` ships with hardcoded listing examples.
- **No backend exists.** Per ADR-004, there is no API to fetch from.
- **Offline must work after first load.** Inline data is trivially available offline; remote fetches would require service-worker caching of fetched JSON, adding complexity.

## Considered Options

1. **None — listing fixtures inline in HTML/JS** (current prototype behavior).
2. **Static JSON file fetched on load** — `listings.json` in the same repo, fetched via `fetch()`.
3. **IndexedDB or localStorage seeded from a JSON file** — persistent local store.

## Decision Outcome

**Chosen option:** "None — listing fixtures inline in HTML/JS", because the prototype already works this way and the POC's purpose is not to demonstrate any data layer. Inlining is also trivially offline-safe.

### Consequences

- Good, because zero work — the prototype's existing data structure is preserved.
- Good, because offline behavior is automatic — there is nothing to fetch.
- Good, because there are no edge cases around fetch failures, race conditions, or cache invalidation of data.
- Bad, because updating listings means editing source — but the POC will be regenerated wholesale if Nora wants different sample data.
- Neutral, because the POC will not test any code path that the full product will reuse — the full product has a real MLS integration per `../specs/001-search-browse/plan.md`.

## Pros and Cons of the Options

### None — inline fixtures (chosen)

- Good, because no work, no failure modes.
- Bad, because does not exercise any real data-loading code path.

### Static JSON file fetched on load

- Good, because separates data from code — easier to swap sample listings.
- Bad, because adds a `fetch()` call that the service worker must cache for offline.
- Bad, because adds failure modes (fetch error, parse error) that the POC does not need.

### IndexedDB / localStorage seeded from JSON

- Good, because would model the persistence path the full product might use.
- Bad, because grossly out of scope for a POC — adds significant complexity for zero validated learning.

## Constraints on Future Work

- Listing data MUST be hardcoded in the source HTML/JS — adding a remote fetch, IndexedDB store, or external JSON file is out of scope and requires a new ADR.
- `localStorage` MAY be used only for transient UI state (e.g., last screen viewed, simulated favorites toggle) — it MUST NOT store listing data or anything that would imply persistence guarantees.
- The POC MUST NOT introduce any data migration, versioning, or schema concept.

## Deferred Aspects

None — the full product's data layer (PostgreSQL 16 + SQLAlchemy + MLS aggregator) is documented in `../specs/001-search-browse/plan.md` and is explicitly out of scope here.
