---
status: accepted
date: 2026-05-20
---

# Service Worker Caching Strategy: Cache-First Same-Origin, Network-First Cross-Origin

## Context and Problem Statement

The POC's service worker must intercept `fetch` events and decide how to serve each request: from the cache, from the network, or some mixture. The choice has direct user-visible consequences: too aggressive a cache means stale fonts and stale shell after edits; too lax a cache means no real offline support, which would defeat the POC's purpose.

The site has two distinct fetch surfaces:

1. **Same-origin requests** — the app shell (`index.html`, `manifest.webmanifest`, two icon PNGs). These rarely change; when they do change, the developer bumps `CACHE_VERSION` to invalidate them deliberately.
2. **Cross-origin requests** — Google Fonts (`fonts.googleapis.com` for CSS, `fonts.gstatic.com` for WOFF2). Versioned by Google; rotation can happen invisibly.

A single global strategy ("cache-first everything" or "network-first everything") would fit one surface poorly. Per-surface strategies cost nothing because there are only two surfaces.

## Decision Drivers

- **Offline-first for the shell.** ADR-002 commits to "shell renders in airplane mode after first load" as the smoke-test pass criterion. The shell strategy must be cache-first.
- **No stale-font surprises.** If Google rotates a font CDN host, cache-first-forever would leave the cached CSS pointing at a dead WOFF2 URL. Network-first defends against this.
- **No build step (ADR-001).** Strategy logic must live in hand-written `service-worker.js`. Anything that requires precomputed cache manifests (Workbox build, hash-based revisioning) is off the table.
- **Simplicity.** This is a POC. A strategy that takes one paragraph to describe is preferable to one that takes a page.

## Considered Options

1. **Cache-first for all requests (single global strategy).**
2. **Network-first for all requests (single global strategy).**
3. **Stale-while-revalidate for all requests.**
4. **Cache-first for same-origin, network-first with cache fallback for cross-origin** (per-surface strategy).
5. **Workbox-generated routes** with precaching + runtime-caching plugins.

## Decision Outcome

**Chosen option:** "Cache-first for same-origin, network-first with cache fallback for cross-origin", because it matches each surface to its actual change profile (shell is developer-controlled and versioned via `CACHE_VERSION`; fonts are externally rotated and need fresh metadata) while staying small enough to fit in a hand-written `service-worker.js` of well under 100 lines.

### Consequences

- Good, because offline works for the shell on first reload after install — the explicit POC pass criterion.
- Good, because Google Font rotations do not produce dead WOFF2 references when the user is online; we still get cache fallback when offline.
- Good, because the entire strategy lives in `~30 lines` of `fetch` handler code — readable in one glance.
- Bad, because the strategy gives no offline guarantee for cross-origin assets the user has never seen online — for fonts this is acceptable (system fallback exists); for any future cross-origin script it would not be.
- Bad, because the two strategies double the cognitive load slightly versus a single global rule. Mitigation: classification is a one-line `request.url.startsWith(self.location.origin)` check.

## Pros and Cons of the Options

### Cache-first everywhere

- Good, because dead simple — one branch.
- Bad, because Google Font CDN rotation produces broken WOFF2 references that persist until a SW version bump.
- Bad, because no way to deliver an updated cross-origin asset without bumping `CACHE_VERSION` — couples cross-origin freshness to shell deploys.

### Network-first everywhere

- Good, because always-fresh.
- Bad, because offline shell support requires explicit cache fallback paths — more conditional logic, not less.
- Bad, because every shell asset request hits the network first even when offline support is the entire point.

### Stale-while-revalidate everywhere

- Good, because users see a fast cached response and background-refresh fixes staleness over time.
- Bad, because precaching is a separate concern that still needs to be set up — SWR alone does not give first-launch offline.
- Bad, because more moving parts (every request fires two operations).
- Bad, because freshness window is unpredictable — users may see stale shell for many sessions until SWR catches up.

### Cache-first same-origin, network-first cross-origin (chosen)

- Good, because matches actual change profiles.
- Good, because trivially small implementation.
- Bad, because two strategies are slightly more complex than one — but only slightly.

### Workbox-generated routes

- Good, because battle-tested patterns, easy to add more surfaces.
- Bad, because Workbox requires a build step (or a CDN-loaded bundle that's ~30KB of code). ADR-001 prohibits the build step; the CDN-load path adds an external dependency and ~30KB to the SW for capabilities the POC does not need.

## Constraints on Future Work

- The service worker MUST classify each `fetch` event by origin and dispatch to exactly one of two strategies: cache-first (same-origin) or network-first with cache fallback (cross-origin).
- Adding a third strategy (e.g., stale-while-revalidate for a new surface) requires a new ADR.
- Any new cross-origin dependency added to the POC MUST be evaluated against the network-first strategy — if it requires guaranteed offline availability, the SW strategy needs to be revisited.
- The cache key MUST be the request URL string (no custom keying — Workbox routing is out of scope per the rejection above).
- The cache name MUST embed a version identifier (currently `nestor-pwa-v1`) so the developer can invalidate stale shell entries by editing `CACHE_VERSION` in the SW source.
- The SW MUST NOT attempt to synthesize fallback responses (e.g., an "offline.html" page) for failed fetches in the POC — failure passes through to the page. Adding a synthesized offline page requires a new ADR.
