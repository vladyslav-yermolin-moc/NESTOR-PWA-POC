# Change: pwa-shell

## Why

The client (Nora) needs to physically install and use the Nestor experience on her iPhone before she will commit to a Progressive Web App as the delivery vehicle for the full Nestor product (per `docs/project-brief.md`). Walkthroughs of the static HTML prototype (`2026-05-19-mvp-showing.html`) on a desktop screen do not let her feel: (a) the install-from-browser flow, (b) the home-screen icon, (c) the standalone launch (no Safari chrome), (d) the offline app-shell behavior after first load.

This change wraps the existing 1191-LOC prototype in the minimum PWA shell that turns it into something Nora can "Add to Home Screen" on iPhone Safari 17+ and Android Chrome 120+. It validates the install experience without changing the prototype's content or UI — the foundational decisions for the POC are locked in ADR-001..006.

Per ADR-001, this is vanilla HTML/CSS/JS with no build step. Per ADR-002, it is a static single-page site. Per ADR-005, it deploys via GitHub Pages.

## What Changes

- **Add `PWA/index.html`**: a copy of `2026-05-19-mvp-showing.html` with the following additions to its `<head>`:
  - `<link rel="manifest" href="manifest.webmanifest">`
  - `<meta name="theme-color" content="#1F2937">` (matches the prototype's dark frame)
  - `<meta name="apple-mobile-web-app-capable" content="yes">` (iOS standalone mode)
  - `<meta name="apple-mobile-web-app-status-bar-style" content="default">`
  - `<meta name="apple-mobile-web-app-title" content="NestorAI">`
  - `<link rel="apple-touch-icon" href="icons/icon-192.png">`
  - A small `<script>` block at the end of `<body>` that registers `service-worker.js` (guarded by `'serviceWorker' in navigator`).
- **Add `PWA/manifest.webmanifest`**: web app manifest with `name: "NestorAI"`, `short_name: "NestorAI"`, `display: "standalone"`, `start_url: "."`, `scope: "."`, `background_color: "#1F2937"`, `theme_color: "#1F2937"`, and two icon entries referencing the PNGs below.
- **Add `PWA/service-worker.js`**: a minimal service worker that:
  - On `install`: pre-caches the app shell (`index.html`, `manifest.webmanifest`, both icons) into a versioned cache (e.g., `nestor-pwa-v1`).
  - On `fetch`: cache-first for same-origin requests, network-first with cache fallback for cross-origin (Google Fonts).
  - On `activate`: deletes stale cache versions.
- **Add `PWA/icons/icon-192.png` and `PWA/icons/icon-512.png`**: two PNGs derived from the prototype's CSS X-in-box logo (192×192 and 512×512, with a dark `#1F2937` background to match `theme_color`).

- **Edit `PWA/2026-05-19-mvp-showing.html`** (the source prototype, now local to `PWA/`): append a single `@media (display-mode: standalone), (display-mode: fullscreen)` block to the existing `<style>` element. The block resets `.phone` to fill the viewport (removing the `box-shadow` ring, the `border-radius`, and the fixed `360×720` dimensions) and sets `html, body { background: #FFFFFF; }` so the installed PWA fills the device screen instead of rendering as a phone-inside-a-phone. The rule is gated by display-mode; the desktop browser view (and `file://` open) is unaffected.

The prototype's structural markup and existing styles are not otherwise modified — the PWA layer (manifest, service worker, head tags, registration script) and the standalone-mode CSS reset are both additive, per ADR-001's constraint that the prototype must still work when opened directly without the service worker registered.

## Scope

### In Scope

- Wrapping the existing prototype as an installable PWA.
- `manifest.webmanifest` with `name`, `short_name`, `icons`, `display: standalone`, `start_url`, `scope`, `background_color`, `theme_color`.
- A basic `service-worker.js` (precache on install + cache-first runtime + cache-version cleanup on activate).
- Required PWA meta tags in `<head>` and the iOS `apple-touch-icon` link.
- Two PNG icons (192px + 512px) generated from the prototype's CSS logo.
- Service-worker registration script inline in `index.html`.
- A standalone-mode CSS reset on `.phone` (and `html, body` background) so the installed PWA fills the device viewport instead of rendering as a phone-inside-a-phone.
- Manual smoke test on a real iPhone (Safari 17+) and a real Android device (Chrome 120+) per ADR-006.

### Out of Scope

- iOS splash screen PNGs (the `apple-touch-startup-image` matrix). White-flash launch on iOS is accepted for the POC; revisit if Nora signals it feels unfinished.
- An on-device "How to install" guidance overlay (iOS lacks `beforeinstallprompt`; Nora will be guided verbally during the demo).
- Push notifications, background sync, periodic sync, Web Share Target.
- Any refactor of the prototype beyond the additive `<head>` tags, SW-registration `<script>`, and the single appended `@media (display-mode: standalone)` CSS block. The prototype's existing markup, screen flow, and JS remain untouched.
- GitHub repo setup, GitHub Pages enablement, and the actual deployment (Vlad will handle locally per the user-stated plan from explore mode).
- Automated tests (Lighthouse CI, Playwright, Vitest) — deferred indefinitely per ADR-006.
- Any backend, API client, auth, or data layer (forbidden by ADR-003 and ADR-004).
- A build step, bundler, or `package.json` (forbidden by ADR-001).

## Impact

**Affected files** (all new, all inside the `PWA/` repo root):

- `2026-05-19-mvp-showing.html` — **modified** (the source prototype, now in `PWA/`); one `@media (display-mode: standalone), (display-mode: fullscreen)` CSS block appended inside the existing `<style>` element (~10 lines, ~15 LOC additional). No other changes.
- `index.html` — new file; byte-identical copy of `PWA/2026-05-19-mvp-showing.html` with PWA `<head>` tags + SW registration script appended.
- `manifest.webmanifest` — new file.
- `service-worker.js` — new file.
- `icons/icon-192.png` — new binary asset.
- `icons/icon-512.png` — new binary asset.

**Third-party CDN dependencies** introduced: none beyond what the prototype already uses (Google Fonts via `fonts.googleapis.com` and `fonts.gstatic.com`). The service worker will treat these as cross-origin and cache them with a network-first strategy.

**No DB migration required** (there is no database — see ADR-003).

**Risks:**

- *GitHub Pages subpath gotcha* — `start_url`, `scope`, and asset paths in the manifest must be relative (`"."` and `"icons/..."`) so they resolve correctly under `<org>.github.io/<repo>/`. ADR-005 calls this out; the design will enforce it.
- *Service-worker registration silently failing* — if the service worker is served from the wrong scope (e.g., a `js/` subdirectory) or over plain HTTP, install will not work and the failure mode is silent. The design will explicitly require root-scope serving and HTTPS (GitHub Pages provides HTTPS by default).
- *iOS first-launch white flash* — without `apple-touch-startup-image` PNGs, iOS shows a white screen briefly on launch. Accepted for the POC (see Out of Scope). If Nora reacts poorly, a follow-up change adds the splash PNGs.
- *Standalone-mode CSS overrides leak to mobile Safari* — the `@media (display-mode: standalone)` block only activates when the page is launched as an installed PWA. Mobile Safari (browser view) still shows the phone-frame chrome. If that also feels weird to Nora when she previews the URL before installing, a follow-up change can either add a viewport-width breakpoint or move the override into a `(max-width: 480px)` media query — out of scope here.

**Team / process impact:**

- Ivan (QA) needs ~10 minutes to run the iPhone + Android smoke test before the URL is shared with Nora.
- Vlad handles the GitHub Pages deployment from a local clone after this change is implemented.
- No other team members are blocked or affected.
