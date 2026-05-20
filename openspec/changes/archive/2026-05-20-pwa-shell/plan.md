# Implementation Plan: pwa-shell

## Overview

Five files need to land in `PWA/`: `index.html` (copied from the prototype with PWA additions), `manifest.webmanifest`, `service-worker.js`, and two icon PNGs. The work splits cleanly into three parallel asset-prep tracks (manifest, service worker, icons) plus one integration step (the `index.html` head + script edits) and one verification step (manual smoke test on real devices).

No database migration. No backend. No build step. No CI pipeline. The output is a folder of five files.

Per ADR-006, the change is not "done" until QA has run the smoke test on a real iPhone (Safari 17+) and a real Android device (Chrome 120+). Per the proposal, Vlad handles GitHub Pages deployment from a local clone after implementation — that step is outside the scope of this change.

References: `proposal.md`, `design.md`, `specs/pwa-shell/spec.md`, `adrs/2026-05-20-service-worker-caching-strategy.md`, and foundational ADRs `docs/decisions/001-006`.

## Implementation Phases

### Phase 1: Seed `PWA/index.html` from the local prototype source

**Changes:**

- The prototype source file `PWA/2026-05-19-mvp-showing.html` already exists in the project (relocated from `../`) and already contains the `@media (display-mode: standalone), (display-mode: fullscreen)` CSS block — that edit was applied as part of this change. Phase 1's job is just to seed `PWA/index.html` as a byte-identical copy of this post-edit source. No further content changes in this phase — PWA additions happen in Phase 5.

**Code sample:**

```bash
cd PWA
cp 2026-05-19-mvp-showing.html index.html
```

**Acceptance Criteria:**

- `PWA/index.html` exists and is byte-identical to `PWA/2026-05-19-mvp-showing.html` (`diff -q` returns no differences).
- Both files contain the `@media (display-mode: standalone), (display-mode: fullscreen)` block inside `<style>`.
- Opening `PWA/index.html` via `file://` in a desktop browser renders the prototype with the desktop phone-frame chrome intact (the `@media` rule does not match in `display-mode: browser`).
- Maps to spec scenarios: **"Prototype Backward Compatibility → file:// opening renders prototype unchanged"** and **"Prototype source diff vs upstream shows only the standalone-mode CSS"**.

**Dependencies:** None.
**Parallel:** Yes — independent of Phases 2, 3, 4.

**Agent Routing:**

- `frontend-dev`: task 1.1 (SEQ within phase — single task).

**Review Checkpoint:**

- Vlad confirms the file copy is in place and matches the source byte-for-byte. Confirms the `@media (display-mode: standalone)` block is present in both files.

---

### Phase 2: Author `manifest.webmanifest`

**Changes:**

- Create `PWA/manifest.webmanifest` with the fields defined in `design.md` → File Changes table.

**Code sample:**

```json
{
  "name": "NestorAI",
  "short_name": "NestorAI",
  "description": "Nestor PWA POC — installable buyer-flow prototype for client review.",
  "start_url": ".",
  "scope": ".",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#1F2937",
  "theme_color": "#1F2937",
  "icons": [
    { "src": "icons/icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable" },
    { "src": "icons/icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ]
}
```

**Acceptance Criteria:**

- File validates against the W3C Web App Manifest spec (pasteable into <https://manifest-validator.appspot.com/> or Chrome DevTools → Application → Manifest, zero errors).
- All path-valued fields (`start_url`, `scope`, icon `src`) are relative — no leading `/`.
- Chrome DevTools → Application → Manifest panel parses the file and previews the 192px icon.
- Maps to spec scenarios: **"Web App Manifest Discoverability → Browser parses manifest successfully"** and **"Manifest paths resolve under GitHub Pages subpath"**.

**Dependencies:** None.
**Parallel:** Yes — independent of Phases 1, 3, 4. Note: this phase's acceptance check inside DevTools depends on Phase 5 having wired the `<link rel="manifest">` tag, but file creation itself is independent.

**Agent Routing:**

- `frontend-dev`: task 2.1 (SEQ within phase — single file).

**Review Checkpoint:**

- Code reviewer verifies: JSON is well-formed; all required keys per spec scenario are present; `display: "standalone"` (not `minimal-ui` or `browser`); colors match the prototype's `#1F2937` theme.

---

### Phase 3: Author `service-worker.js`

**Changes:**

- Create `PWA/service-worker.js` implementing the three event handlers (`install`, `activate`, `fetch`) described in `design.md` and constrained by `adrs/2026-05-20-service-worker-caching-strategy.md`.

**Code sample:**

```javascript
const CACHE_VERSION = 'nestor-pwa-v1';

const APP_SHELL = [
  './',
  'index.html',
  'manifest.webmanifest',
  'icons/icon-192.png',
  'icons/icon-512.png',
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_VERSION).then((cache) => cache.addAll(APP_SHELL)),
  );
  self.skipWaiting();
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((names) =>
      Promise.all(
        names
          .filter((name) => name !== CACHE_VERSION)
          .map((name) => caches.delete(name)),
      ),
    ),
  );
  self.clients.claim();
});

self.addEventListener('fetch', (event) => {
  const { request } = event;
  if (request.method !== 'GET') return;

  const isSameOrigin = new URL(request.url).origin === self.location.origin;

  if (isSameOrigin) {
    // cache-first
    event.respondWith(
      caches.match(request).then((cached) => {
        if (cached) return cached;
        return fetch(request).then((response) => {
          if (response.ok) {
            const copy = response.clone();
            caches.open(CACHE_VERSION).then((cache) => cache.put(request, copy));
          }
          return response;
        });
      }),
    );
  } else {
    // network-first with cache fallback
    event.respondWith(
      fetch(request)
        .then((response) => {
          if (response.ok) {
            const copy = response.clone();
            caches.open(CACHE_VERSION).then((cache) => cache.put(request, copy));
          }
          return response;
        })
        .catch(() => caches.match(request)),
    );
  }
});
```

**Acceptance Criteria:**

- Chrome DevTools → Application → Service Workers shows the SW as "activated and running" after first load on a localhost HTTPS reverse proxy or after Pages deploy.
- After first load, Chrome DevTools → Application → Cache Storage shows `nestor-pwa-v1` with the four expected shell entries.
- Toggling "Offline" in DevTools and reloading the page still serves the shell.
- Maps to spec scenarios: **"Service Worker Registration → Service worker registers on first visit"**, **"App Shell Precaching → Install event precaches the shell"**, **"Stale Cache Cleanup on Activate → Activate purges stale cache versions"**, **"Offline Cache-First Runtime → Installed PWA renders the shell while offline"**, **"Network-First with Cache Fallback → Font CSS served from cache when offline"**.

**Dependencies:** None for authoring; verification of "precaches the shell" requires Phases 4 (icons) and 5 (registration) so that the SW can actually fetch all four shell entries.

**Parallel:** Yes — file authoring is independent of Phases 1, 2, 4.

**Agent Routing:**

- `frontend-dev`: task 3.1 (SEQ within phase — single file).

**Review Checkpoint:**

- Code reviewer verifies: `CACHE_VERSION` constant present; `APP_SHELL` array contains exactly the four expected entries; `install` and `activate` are wrapped in `event.waitUntil(...)`; cross-origin branch uses network-first per ADR; no synthesized offline fallback is introduced.

---

### Phase 4: Generate `icons/icon-192.png` and `icons/icon-512.png`

**Changes:**

- Create `PWA/icons/icon-192.png` (192×192) and `PWA/icons/icon-512.png` (512×512) — both depicting the prototype's X-in-box CSS logo on a `#1F2937` background with white strokes.

**Code sample:**

The prototype's logo is drawn in CSS via two pseudo-elements forming an X inside a square border. To rasterize as a PNG, the simplest path is to render the existing CSS in a browser at the target size and screenshot, or to author the same shape directly in any vector tool (Figma, Inkscape, Sketch) and export at 192 + 512.

For the throwaway POC, the fastest path is an SVG that mirrors the CSS:

```html
<!-- icon-source.svg — render via browser and export PNGs at 192 & 512 -->
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 192 192">
  <rect width="192" height="192" fill="#1F2937"/>
  <rect x="48" y="48" width="96" height="96" fill="none" stroke="#FFFFFF" stroke-width="6"/>
  <line x1="51" y1="51" x2="141" y2="141" stroke="#FFFFFF" stroke-width="6"/>
  <line x1="141" y1="51" x2="51" y2="141" stroke="#FFFFFF" stroke-width="6"/>
</svg>
```

Open in Chrome, screenshot at 192×192 → save as `icons/icon-192.png`. Re-render at 512×512 viewport (scale `viewBox` accordingly) → save as `icons/icon-512.png`. Both PNGs MUST be opaque (no transparency around the dark square — the `purpose: "any maskable"` declaration tells Android to crop if needed).

**Acceptance Criteria:**

- `PWA/icons/icon-192.png` exists, exactly 192×192 pixels, opaque PNG.
- `PWA/icons/icon-512.png` exists, exactly 512×512 pixels, opaque PNG.
- Both files preview correctly in Chrome DevTools → Application → Manifest panel.
- Maps to spec scenario: **"iOS launches the installed app in standalone mode → home-screen icon visual MUST be `icons/icon-192.png`"** (depends on Phases 2 + 5 wiring to be end-to-end verified).

**Dependencies:** None for asset creation.
**Parallel:** Yes — independent of Phases 1, 2, 3.

**Agent Routing:**

- `designer`: tasks 4.1, 4.2 (SEQ within phase — render then export). The asset creation is light enough that `frontend-dev` can also handle it if no designer is available.

**Review Checkpoint:**

- Visual review: PNGs look like the prototype logo at both sizes; backgrounds are `#1F2937`; strokes are clean (no anti-aliasing fuzz at corners). Reviewer opens both files at native size to confirm.

---

### Phase 5: Wire PWA `<head>` tags + SW registration in `index.html`

**Changes:**

- Edit `PWA/index.html` to add the PWA-specific `<head>` tags and the SW-registration `<script>` near `</body>`.

**Code sample:**

Insert into `<head>` (after the existing `<meta name="viewport">`, before the existing `<link>` to Google Fonts):

```html
<link rel="manifest" href="manifest.webmanifest">
<meta name="theme-color" content="#1F2937">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="apple-mobile-web-app-title" content="NestorAI">
<link rel="apple-touch-icon" href="icons/icon-192.png">
```

Insert just before `</body>`:

```html
<script>
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('service-worker.js').catch(() => {
        // Silent failure — file:// or unsupported context. POC continues without offline.
      });
    });
  }
</script>
```

No other edits to `index.html` are permitted in this change.

**Acceptance Criteria:**

- `diff PWA/index.html 2026-05-19-mvp-showing.html` shows exactly two additions: the six `<head>` lines and the `<script>` block before `</body>`.
- Opening `PWA/index.html` via `file://` still renders the prototype identically (the registration call rejects, the `.catch(() => {})` swallows it, no console errors disrupt UX).
- Maps to spec scenarios: **"PWA-Specific Meta Tags for iOS → iOS launches the installed app in standalone mode"**, **"Service Worker Registration → Service worker is silently skipped over file://"**, **"Prototype Backward Compatibility → Prototype content is byte-identical except for the added head + script"**.

**Dependencies:** Phase 1 (source file must exist). Soft dependencies on Phases 2, 3, 4 (the referenced files should exist for the page to fully function, but the edits to `index.html` can happen before those files land — the `.catch()` will swallow any 404s).

**Parallel:** No — touches the file produced by Phase 1.

**Agent Routing:**

- `frontend-dev`: tasks 5.1, 5.2 (SEQ — both edits to the same file).

**Review Checkpoint:**

- Code reviewer diffs `PWA/index.html` against `2026-05-19-mvp-showing.html` and confirms exactly the two additions specified above, in the correct locations. No other lines changed.

---

### Phase 6: Manual smoke test on real iPhone + Android (gate before sharing URL)

**Changes:**

- Run the manual smoke test per ADR-006 on a real iPhone (Safari 17+) and a real Android device (Chrome 120+). Document the result inline as a checklist comment on the PR or in `openspec/changes/pwa-shell/notes.md` if no PR exists.

**Code sample:**

```text
Smoke-test checklist (record date, device, OS, browser, result):

iPhone (Safari 17+):
  [ ] Open URL → page renders without errors
  [ ] Mobile Safari view: phone-frame chrome IS visible (display-mode: browser)
  [ ] Tap Share → "Add to Home Screen" appears in the share sheet
  [ ] Confirm → home-screen icon appears with label "NestorAI"
  [ ] Tap home-screen icon → standalone launch (no Safari address bar)
  [ ] Installed view: phone-frame chrome IS GONE (display-mode: standalone)
  [ ] `.phone` content fills the device viewport (no inner ring or radius)
  [ ] Enable Airplane Mode → re-launch icon → shell still renders
  [ ] Tap through 3 prototype screens → all render correctly

Android (Chrome 120+):
  [ ] Open URL → page renders without errors
  [ ] Chrome browser view: phone-frame chrome IS visible
  [ ] Chrome surfaces install affordance (install card or menu item)
  [ ] Install → home-screen icon appears with label "NestorAI"
  [ ] Tap icon → splash with #1F2937 background + 512 icon → standalone launch
  [ ] Installed view: phone-frame chrome IS GONE; content fills viewport
  [ ] Enable Airplane Mode → re-launch icon → shell still renders
  [ ] Tap through 3 prototype screens → all render correctly
```

**Acceptance Criteria:**

- All checkboxes above pass on both platforms before the URL is shared with Nora (hard gate per the proposal's smoke-test-before-share rule and per ADR-006's constraints). The phone-frame-in-standalone-mode check is a new gate added in this revision.
- Test execution requires the site to be served over HTTPS — the tester deploys to a local clone of GitHub Pages (or any HTTPS preview host) before running.
- Maps to all spec scenarios end-to-end (full acceptance integration).

**Dependencies:** Phases 1–5 all complete.
**Parallel:** No — verification is the last step and gates handoff to Vlad.

**Agent Routing:**

- `tester`: tasks 6.1, 6.2 (SEQ — iPhone first, then Android). Ivan (project QA) is the human assignee.

**Review Checkpoint:**

- Vlad reviews the completed checklist before sharing the URL with Nora. Any failing item blocks the share.

---

## Parallelization Notes

- **Wave 1 (parallel):** Phase 1 (copy HTML), Phase 2 (manifest), Phase 3 (service worker), Phase 4 (icons). All four are independent file-creation tasks.
- **Wave 2 (sequential, after Wave 1):** Phase 5 (edit `index.html` to wire everything together). Depends on Phase 1 (the file must exist) and the existence of the files from Phases 2/3/4 to be end-to-end runnable.
- **Wave 3 (sequential, after Wave 2):** Phase 6 (smoke test). Hard gate before sharing the URL.

**Database migration:** N/A — there is no database in this project (per ADR-003).

**Rollback strategy:** If any phase fails verification, revert the affected file(s) with `git checkout -- <path>`. Because there is no build step and no remote infrastructure, rollback is purely a local-file operation. Once the URL is shared with Nora, "rollback" means deploying a previous Pages commit — but that is outside the scope of this change (Vlad owns Pages deployment).
