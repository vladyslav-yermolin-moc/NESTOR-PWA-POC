# Bootstrap Assessment

- **Date:** 2026-05-19
- **Project state:** new
- **Project brief:** docs/project-brief.md

## Project Brief

See [docs/project-brief.md](../docs/project-brief.md).

Summary: Nestor PWA POC — a small proof-of-concept to give the client (Nora) the ability to install and test PWA technology on her own device, using the existing `2026-05-19-mvp-showing.html` prototype as the starting payload. In-scope: PWA install flow + static listing cards + mobile-first UI. Out of scope: AI chat, auth, favorites persistence, production infra.

## External Documentation

Paths to include as read-only context for all artifact generation:

- docs/

## Foundational Decisions

### ADR-001: Tech Stack

- **Status:** accepted
- **Decision:** Vanilla HTML + CSS + JavaScript (no framework, no build step). Starting payload is the existing clickable prototype at `2026-05-19-mvp-showing.html` (~1200 LOC, mobile-first phone-frame UI). PWA layer added via a `manifest.webmanifest` and a `service-worker.js` (Cache-First strategy for the app shell). Static assets served as-is.
- **Constraints on future work:**
  - All new code MUST be written as vanilla HTML/CSS/JS — no React, Vue, Svelte, or other framework dependencies.
  - There MUST NOT be a bundler or transpiler step (no Vite, Webpack, esbuild, TS compiler) — files are served as-authored.
  - The PWA layer (`manifest.webmanifest`, `service-worker.js`) MUST be additive — the prototype HTML must remain functional when opened directly without the service worker registered.
  - All third-party dependencies MUST be loaded via `<link>` / `<script>` CDN tags (e.g., Google Fonts) — no npm install.
  - JavaScript MUST target evergreen mobile browsers (iOS Safari 17+, Chrome 120+) — no transpilation, but no IE / legacy fallbacks either.

### ADR-002: Architecture Pattern

- **Status:** accepted
- **Decision:** Static single-page HTML site. One `index.html` (forked from the prototype) plus a `manifest.webmanifest`, a `service-worker.js`, and an `icons/` directory. No backend, no routing layer, no API calls. All screen navigation is client-side via JS show/hide on a single document.
- **Constraints on future work:**
  - The site MUST be deployable as a folder of static files — no server-side rendering, no runtime build.
  - There MUST be exactly one HTML entry point (`index.html`); multi-page architecture is not introduced in the POC.
  - Service-worker scope MUST be the site root (`/`) so install + offline cover the entire app shell.
  - State (current screen, simulated favorites, etc.) MUST live in-memory or in `localStorage` only — no remote persistence.

### ADR-003: Data Storage

- **Status:** accepted
- **Decision:** No data storage. All listing fixtures are inline in the HTML/JS (as in the existing prototype). No database, no fetch from external APIs, no localStorage for listing data.
- **Constraints on future work:**
  - Listing data MUST be hardcoded in the source file — adding a JSON fetch, indexedDB, or remote endpoint is out of scope and requires a new ADR.
  - `localStorage` MAY be used only for transient UI state (e.g., last screen viewed) — not for listing data.

### ADR-004: API Style and Auth

- **Status:** accepted
- **Decision:** No API and no authentication. The POC has no server-side component and no concept of a user account.
- **Constraints on future work:**
  - The POC MUST NOT make any outbound HTTP calls except for CDN-hosted static assets (fonts, icons).
  - Adding auth, sessions, or any API client requires a new change proposal that promotes the project past the POC stage.

### ADR-005: Deployment Model

- **Status:** accepted
- **Decision:** GitHub Pages. The POC is published as a static site from a GitHub repo with Pages enabled, served over HTTPS at a `https://<org>.github.io/<repo>/` URL that is shared with Nora. HTTPS is required for service worker registration on real devices.
- **Constraints on future work:**
  - The build artifact MUST be a folder of static files committed to the repo (no GitHub Action build step required for the POC).
  - The URL given to Nora MUST be HTTPS (GitHub Pages provides this by default) — service worker registration fails on plain HTTP.
  - Custom domain is out of scope; the default `*.github.io` URL is acceptable for the POC.
  - All paths in `manifest.webmanifest` and the service worker MUST be relative or scoped to the repo subpath — no assumptions about being served from the root.

### ADR-006: Testing Strategy

- **Status:** accepted
- **Decision:** Manual smoke testing on a real iPhone (Safari) and a real Android device (Chrome). The test passes when: (1) the PWA installs from the browser's "Add to Home Screen" prompt, (2) the branded splash screen appears on launch, (3) all prototype screens render and navigation works offline after first load. No automated tests (Vitest, Playwright, Lighthouse CI) are introduced for the POC.
- **Constraints on future work:**
  - Before sharing the URL with Nora, the install flow MUST be smoke-tested on at least one iPhone (Safari 17+) and one Android device (Chrome 120+).
  - Each merged change to the POC MUST be re-smoke-tested on iPhone before re-sharing.
  - Automated tests MAY be added if the POC scope grows beyond a clickable demo — that decision will create a new ADR.
