---
status: accepted
date: 2026-05-19
category: architecture
---

# ADR-002: Architecture Pattern — Static Single-Page Site

## Context and Problem Statement

The POC needs an architecture shape that supports (a) installable PWA behavior on iPhone Safari and Android Chrome, (b) the existing prototype's multi-screen UI driven entirely by client-side JS show/hide on a single document, and (c) zero backend. The decision is how to organize the static files and the navigation model.

## Decision Drivers

- **Single-document prototype.** The existing `2026-05-19-mvp-showing.html` already implements all "screens" as `.screen` divs inside one document, toggled by `display: none/block`. Splitting into multiple HTML pages would require restructuring the working artifact.
- **Simplest service-worker scope.** One HTML entry point means one cache namespace, one fetch handler, no cross-route considerations.
- **Static deployability.** The site must be hostable on GitHub Pages — a folder of static files with no server-side runtime.

## Considered Options

1. **Static single-page site (one `index.html`)** — one HTML entry point, client-side screen toggling, manifest + service worker at root.
2. **Static multi-page site** — separate HTML files per screen (`index.html`, `chat.html`, `details.html`), linked with anchor tags.
3. **Client-routed SPA on a framework** — Vite/React with react-router-dom (rejected in ADR-001).

## Decision Outcome

**Chosen option:** "Static single-page site (one `index.html`)", because the existing prototype already follows this pattern and the POC's navigation needs (4–6 screens shown/hidden) do not warrant either separate HTML pages or a router. Service-worker registration scope matches the document scope, simplifying cache invalidation.

### Consequences

- Good, because preserves the existing prototype's structure verbatim — no porting required.
- Good, because service-worker scope is trivially the entire app (one document, one cache).
- Good, because the entire site is two-to-five files (`index.html`, `manifest.webmanifest`, `service-worker.js`, optional `icons/` folder, optional split CSS/JS).
- Bad, because there are no per-screen URLs — deep linking and browser back-button behavior across screens are limited.
- Bad, because all markup loads at once, even for unseen screens — but the prototype is small enough that this is acceptable.
- Neutral, because state lives in JS variables and (optionally) `localStorage` — no remote persistence layer needed for the POC.

## Pros and Cons of the Options

### Static single-page site (chosen)

- Good, because zero structural change from the existing prototype.
- Good, because simplest possible PWA shell.
- Bad, because no native browser routing.

### Static multi-page site

- Good, because each screen gets a real URL.
- Bad, because requires extracting each screen into its own HTML file — non-trivial refactor.
- Bad, because the service worker has to cache and route across multiple documents.

### Client-routed SPA on a framework

- Rejected in ADR-001 (framework cost not justified for a POC).

## Constraints on Future Work

- The site MUST be deployable as a folder of static files — no server-side rendering, no runtime build step.
- There MUST be exactly one HTML entry point (`index.html`) in the POC. Adding additional HTML entry points requires a new ADR.
- The service-worker scope MUST be the site root (`/` or the GitHub Pages subpath) so install and offline behavior cover the entire app shell.
- State (current screen, simulated favorites, etc.) MUST live in memory or in `localStorage` only — no remote persistence is added in the POC.
- Client-side navigation MUST work without changing the document URL (the existing show/hide model). Introducing the History API or hash-based routing requires a new ADR.

## Deferred Aspects

None — POC-scoped.
