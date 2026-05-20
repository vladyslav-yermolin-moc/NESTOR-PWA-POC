# Tasks

## Phase 1: Seed `PWA/index.html` from the local prototype source

- [x] 1.1 Copy `PWA/2026-05-19-mvp-showing.html` (which already contains the `@media (display-mode: standalone)` block) to `PWA/index.html` byte-identical — [plan §Phase 1] / [spec: Prototype Backward Compatibility → file:// opening renders prototype unchanged + Prototype source diff vs upstream shows only the standalone-mode CSS]

## Phase 2: Author `manifest.webmanifest`

- [x] 2.1 Create `PWA/manifest.webmanifest` with name=NestorAI, short_name=NestorAI, start_url=".", scope=".", display=standalone, theme/background_color=#1F2937, and 192+512 icon entries — [plan §Phase 2] / [spec: Web App Manifest Discoverability → Browser parses manifest successfully + Manifest paths resolve under GitHub Pages subpath]

## Phase 3: Author `service-worker.js`

- [x] 3.1 Create `PWA/service-worker.js` with CACHE_VERSION constant, APP_SHELL precache list, install/activate/fetch handlers, cache-first for same-origin, network-first with cache fallback for cross-origin — [plan §Phase 3] / [spec: Service Worker Registration + App Shell Precaching + Stale Cache Cleanup on Activate + Offline Cache-First Runtime + Network-First with Cache Fallback]

## Phase 4: Generate icons

- [x] 4.1 Author `icon-source.svg` mirroring the prototype's X-in-box CSS logo on #1F2937 — [plan §Phase 4]
- [x] 4.2 Export `PWA/icons/icon-192.png` (192×192, opaque PNG) and `PWA/icons/icon-512.png` (512×512, opaque PNG) — [plan §Phase 4] / [spec: PWA-Specific Meta Tags for iOS → iOS launches the installed app in standalone mode]

## Phase 5: Wire PWA `<head>` tags + SW registration in `index.html`

- [x] 5.1 Insert the six PWA `<head>` lines (manifest link, theme-color, four iOS meta/link tags) into `PWA/index.html` — [plan §Phase 5] / [spec: PWA-Specific Meta Tags for iOS + Web App Manifest Discoverability]
- [x] 5.2 Insert the feature-detected `serviceWorker.register('service-worker.js')` `<script>` block just before `</body>` in `PWA/index.html` — [plan §Phase 5] / [spec: Service Worker Registration → Service worker registers on first visit + silently skipped over file://]

## Phase 6: Manual smoke test on real iPhone + Android

- [ ] 6.1 Smoke-test on iPhone Safari 17+: open URL, confirm phone-frame visible in browser view, Share → Add to Home Screen, launch from home screen, confirm phone-frame GONE in standalone view, content fills viewport, reload offline, navigate 3 prototype screens — [plan §Phase 6] / [spec: all scenarios end-to-end, especially Standalone-Mode Viewport Adaptation]
- [ ] 6.2 Smoke-test on Android Chrome 120+: open URL, confirm phone-frame visible in browser view, install via auto-prompt, launch from home screen (splash with #1F2937 + 512 icon), confirm phone-frame GONE in standalone view, reload offline, navigate 3 prototype screens — [plan §Phase 6] / [spec: all scenarios end-to-end]
- [ ] 6.3 Record results in PR/notes and confirm gate before sharing URL with Nora — [plan §Phase 6]
