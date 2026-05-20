# PWA Shell Specification

This specification defines the behavioral contract for the Nestor PWA POC shell — the manifest, service worker, head tags, icons, and standalone-mode CSS that turn the existing HTML prototype into an installable Progressive Web App on iPhone Safari 17+ and Android Chrome 120+.

## Requirements

### Requirement: Web App Manifest Discoverability

The application MUST expose a valid `manifest.webmanifest` from the `<head>` of `index.html` so that browsers can recognize the site as a Progressive Web App and surface the install affordance (Chrome's auto-install card, Safari's "Add to Home Screen" eligibility).

The manifest MUST declare `name`, `short_name`, `start_url`, `scope`, `display: "standalone"`, `background_color`, `theme_color`, and an `icons` array with at least one 192×192 PNG and one 512×512 PNG. All path-valued fields (`start_url`, `scope`, icon `src` values) MUST be relative paths so the site works under both `http://localhost/` and `https://<org>.github.io/<repo>/`.

#### Scenario: Browser parses manifest successfully

- **GIVEN** Nora opens the deployed POC URL in Chrome 120+ on a phone
- **WHEN** the browser parses `index.html`
- **THEN** it MUST find a `<link rel="manifest" href="manifest.webmanifest">` element in the `<head>`
- **AND** the manifest MUST validate against the W3C Web App Manifest spec
- **AND** Chrome MUST consider the site installable (`beforeinstallprompt` fires or the browser surfaces an install UI)

#### Scenario: Manifest paths resolve under GitHub Pages subpath

- **GIVEN** the site is served from `https://<org>.github.io/nestor-pwa-poc/`
- **WHEN** the browser fetches `manifest.webmanifest` and resolves its `start_url`, `scope`, and icon `src` fields
- **THEN** every resolved URL MUST be under `https://<org>.github.io/nestor-pwa-poc/`
- **AND** no resolved URL MUST resolve to the GitHub Pages domain root (`https://<org>.github.io/`)

#### Scenario: Manifest fields missing

- **GIVEN** an attempt to ship the POC without one of: `name`, `start_url`, `display`, or both required icons
- **WHEN** the smoke-test pre-share check runs
- **THEN** the check MUST fail
- **AND** the missing field(s) MUST be reported
- **AND** the URL MUST NOT be shared with Nora until corrected

---

### Requirement: Service Worker Registration

The application MUST register a service worker at `service-worker.js` from the same origin and scope as `index.html`. Registration MUST be feature-detected (guarded by `'serviceWorker' in navigator`) and MUST NOT block first paint of the prototype content. Failure to register (unsupported context, e.g., `file://` or HTTP) MUST be silent and non-fatal — the prototype content MUST still render correctly.

#### Scenario: Service worker registers on first visit (HTTPS)

- **GIVEN** Nora visits the POC over HTTPS in a browser that supports service workers
- **WHEN** the `load` event fires on `window`
- **THEN** `navigator.serviceWorker.register('service-worker.js')` MUST be called with no leading slash
- **AND** within 5 seconds, `navigator.serviceWorker.controller` SHOULD be non-null on subsequent navigations
- **AND** the SW's `scope` MUST equal the directory containing `index.html`

#### Scenario: Service worker is silently skipped over `file://`

- **GIVEN** a developer opens `PWA/index.html` directly in a browser via `file://`
- **WHEN** the page loads
- **THEN** the registration code MUST detect the unsupported context (registration promise rejects)
- **AND** the rejection MUST be caught silently
- **AND** the prototype content MUST render and remain interactive exactly as it did before this change

#### Scenario: Service worker is unavailable in a legacy browser

- **GIVEN** a hypothetical browser without `navigator.serviceWorker`
- **WHEN** the page loads
- **THEN** the registration code MUST short-circuit on the feature check
- **AND** no error MUST be thrown
- **AND** the prototype MUST remain fully functional (no offline support, but no degradation either)

---

### Requirement: App Shell Precaching

On the service worker's `install` event, the SW MUST precache the app shell into a versioned cache. The precached set MUST include exactly: `index.html`, `manifest.webmanifest`, `icons/icon-192.png`, and `icons/icon-512.png`. The cache name MUST contain a version identifier (e.g., `nestor-pwa-v1`) so future shell changes can invalidate stale entries by bumping the version.

#### Scenario: Install event precaches the shell

- **GIVEN** the SW is registered for the first time
- **WHEN** the SW's `install` event fires
- **THEN** the SW MUST open a cache named with the current `CACHE_VERSION`
- **AND** the SW MUST add all four shell assets to the cache via `cache.addAll([...])`
- **AND** the install MUST succeed only if all four assets are successfully cached

#### Scenario: Install fails if a shell asset is missing

- **GIVEN** `icons/icon-192.png` is missing from the deployed repo
- **WHEN** the SW's `install` event fires
- **THEN** `cache.addAll([...])` MUST reject
- **AND** the SW install MUST fail
- **AND** the failed install MUST NOT replace any previously activated SW version

---

### Requirement: Stale Cache Cleanup on Activate

On `activate`, the SW MUST enumerate all cache names via `caches.keys()` and delete any that do not match the current `CACHE_VERSION`. This guarantees that when the developer bumps the version (e.g., from `v1` to `v2`) on a future shell update, the prior cache is purged on the next activation rather than persisting indefinitely.

#### Scenario: Activate purges stale cache versions

- **GIVEN** a previous SW deployment created caches named `nestor-pwa-v1` and `nestor-pwa-temp`
- **AND** the current SW's `CACHE_VERSION` is `nestor-pwa-v2`
- **WHEN** the new SW's `activate` event fires
- **THEN** the SW MUST iterate `caches.keys()`
- **AND** the SW MUST call `caches.delete(name)` for every name other than `nestor-pwa-v2`
- **AND** after activation, only `nestor-pwa-v2` MUST remain

#### Scenario: Activate is a no-op when no stale caches exist

- **GIVEN** the SW is being activated for the first time and `caches.keys()` returns only `nestor-pwa-v1`
- **WHEN** the `activate` event fires
- **THEN** the SW MUST NOT delete the current cache
- **AND** the activation MUST complete successfully

---

### Requirement: Offline Cache-First Runtime for Same-Origin Requests

After the first successful load, the SW MUST intercept same-origin fetch requests and serve them from the cache when available (cache-first strategy). When a same-origin request misses the cache, the SW MUST fall back to network and SHOULD populate the cache with the network response for future offline use.

#### Scenario: Installed PWA renders the shell while offline

- **GIVEN** Nora has successfully installed the PWA and used it once online
- **AND** Nora's phone is in airplane mode
- **WHEN** Nora launches the PWA from the home-screen icon
- **THEN** the SW MUST serve `index.html`, `manifest.webmanifest`, and both icons from the cache
- **AND** the prototype's app shell MUST render
- **AND** all in-shell client-side screen navigation MUST work

#### Scenario: Same-origin asset not in cache, network available

- **GIVEN** a same-origin request is made for an asset not present in the cache (hypothetical future asset)
- **AND** the network is reachable
- **WHEN** the SW intercepts the request
- **THEN** the SW MUST `fetch()` the asset from the network
- **AND** the SW SHOULD store the response in the current versioned cache
- **AND** the response MUST be returned to the page

#### Scenario: Same-origin asset not in cache, offline

- **GIVEN** a same-origin request for an uncached asset
- **AND** the network is unreachable
- **WHEN** the SW intercepts the request
- **THEN** the network fetch MUST fail
- **AND** the SW MUST return the failed response to the page (no synthetic fallback in the POC)

---

### Requirement: Network-First with Cache Fallback for Cross-Origin Requests

Cross-origin requests (specifically Google Fonts at `fonts.googleapis.com` and `fonts.gstatic.com`, the only cross-origin dependency in the prototype) MUST be served network-first with a cache fallback. The SW MUST attempt the network fetch first; on success it MUST cache the response and return it; on failure (offline) it MUST attempt to return a cached response.

#### Scenario: Font CSS served from network when online

- **GIVEN** Nora opens the PWA online
- **WHEN** the page requests `https://fonts.googleapis.com/css2?family=Inter:...`
- **THEN** the SW MUST attempt a network fetch first
- **AND** on success, the SW MUST store the response in the cache
- **AND** the response MUST be returned to the page

#### Scenario: Font CSS served from cache when offline

- **GIVEN** Nora has previously loaded the PWA online (fonts are cached)
- **AND** Nora's phone is now in airplane mode
- **WHEN** the page requests the same font URL
- **THEN** the network fetch MUST fail
- **AND** the SW MUST return the cached response
- **AND** the prototype's typography MUST render using Inter (not the fallback system font)

---

### Requirement: PWA-Specific Meta Tags for iOS

The `<head>` of `index.html` MUST include the iOS-specific PWA meta tags so that "Add to Home Screen" on iOS Safari 17+ produces a standalone-mode launch (no Safari chrome) with the correct title and icon. The required tags are:

- `<meta name="apple-mobile-web-app-capable" content="yes">`
- `<meta name="apple-mobile-web-app-status-bar-style" content="default">`
- `<meta name="apple-mobile-web-app-title" content="NestorAI">`
- `<link rel="apple-touch-icon" href="icons/icon-192.png">`

#### Scenario: iOS launches the installed app in standalone mode

- **GIVEN** Nora has added the POC to her iPhone home screen via Share → Add to Home Screen
- **WHEN** Nora taps the home-screen icon
- **THEN** the app MUST launch in a standalone window
- **AND** the Safari address bar MUST NOT be visible
- **AND** the home-screen icon label MUST read "NestorAI"
- **AND** the home-screen icon visual MUST be `icons/icon-192.png` (auto-scaled by iOS)

#### Scenario: iOS meta tag missing

- **GIVEN** `<meta name="apple-mobile-web-app-capable" content="yes">` is absent from `index.html`
- **WHEN** Nora adds the site to her home screen and taps the icon
- **THEN** the site MUST open inside Safari with the address bar visible (not standalone)
- **AND** the smoke-test pre-share check MUST detect this and fail

---

### Requirement: HTTPS Serving for Service Worker Eligibility

The deployed POC MUST be served over HTTPS (the default for GitHub Pages on `*.github.io` URLs). Service worker registration over plain HTTP (except `localhost`) MUST NOT be relied upon.

#### Scenario: HTTPS deployment registers the SW

- **GIVEN** the site is published at `https://<org>.github.io/<repo>/`
- **WHEN** Nora visits the URL
- **THEN** the SW MUST register successfully
- **AND** the install card / "Add to Home Screen" affordance MUST appear

#### Scenario: HTTP serving prevents SW registration

- **GIVEN** the site is hypothetically served over plain HTTP (not the actual deployment, but a misconfiguration scenario)
- **WHEN** a browser attempts SW registration
- **THEN** the registration MUST fail (per spec, SWs require secure contexts)
- **AND** the silent failure handler in `index.html` MUST swallow the error
- **AND** the page MUST still render the prototype content

---

### Requirement: Prototype Backward Compatibility

The prototype HTML's existing markup, screen flow, and inline JavaScript MUST NOT be modified by this change. The only permitted edits to the prototype source file (`PWA/2026-05-19-mvp-showing.html`) are: (a) one `@media (display-mode: standalone), (display-mode: fullscreen)` CSS block appended inside the existing `<style>` element. The only permitted additions in `PWA/index.html` beyond the prototype source are: (a) PWA `<head>` tags, (b) the SW-registration `<script>` near `</body>`. Opening either file directly via `file://` MUST render the prototype exactly as the original (un-edited) file does — the `@media (display-mode: standalone)` block does not activate in browser-tab mode.

#### Scenario: file:// opening renders prototype unchanged

- **GIVEN** a developer opens `PWA/index.html` via `file:///.../PWA/index.html`
- **WHEN** the page loads
- **THEN** all original prototype screens MUST render with the desktop phone-frame chrome intact (the `@media (display-mode: standalone)` block MUST NOT activate)
- **AND** all original prototype interactions (screen navigation, etc.) MUST continue to work
- **AND** no SW-related console error MUST disrupt the user (registration error MUST be silently caught)

#### Scenario: Prototype diff vs `index.html` shows only the PWA additions

- **GIVEN** a diff of `PWA/index.html` against `PWA/2026-05-19-mvp-showing.html` (the post-edit source)
- **WHEN** the diff is reviewed
- **THEN** the only changes MUST be: additions inside `<head>` (manifest link, theme-color meta, four iOS meta/link tags) and a single `<script>` block appended just before `</body>`
- **AND** no existing `.screen`, `.phone`, or prototype-content CSS rule MUST be altered
- **AND** no original `<script>` or inline JS MUST be altered or relocated

#### Scenario: Prototype source diff vs upstream shows only the standalone-mode CSS

- **GIVEN** a diff of `PWA/2026-05-19-mvp-showing.html` against the upstream `../2026-05-19-mvp-showing.html` at the parent project root
- **WHEN** the diff is reviewed
- **THEN** the only change MUST be a single `@media (display-mode: standalone), (display-mode: fullscreen)` block appended inside `<style>` (containing `.phone { width: 100%; height: 100%; border-radius: 0; box-shadow: none; }` and `html, body { background: #FFFFFF; }`)
- **AND** no other CSS rule MUST be altered

---

### Requirement: Standalone-Mode Viewport Adaptation

When the page is launched in `display-mode: standalone` or `display-mode: fullscreen` (i.e., from a home-screen installation on iOS Safari 17+ or Android Chrome 120+), the prototype's `.phone` element MUST fill the device viewport and the desktop phone-frame chrome MUST be absent. When the page is loaded in a regular browser tab (`display-mode: browser`), the phone-frame chrome MUST remain visible.

#### Scenario: Installed PWA fills the viewport on iPhone

- **GIVEN** Nora has installed the PWA to her iPhone home screen
- **WHEN** Nora taps the home-screen icon to launch the app
- **THEN** the `@media (display-mode: standalone)` rule MUST match
- **AND** the `.phone` element MUST span the full device width and height (no 360×720 fixed dimensions visible)
- **AND** the dark `box-shadow` ring around `.phone` MUST NOT be visible
- **AND** the `border-radius: 32px` MUST NOT be visible (the content MUST extend to the device's corners modulo iOS safe-area)

#### Scenario: Mobile Safari browser view preserves the desktop phone-frame

- **GIVEN** Nora opens the GitHub Pages URL in mobile Safari (browser tab, before install)
- **WHEN** the page renders
- **THEN** the page MUST be in `display-mode: browser`
- **AND** the `@media (display-mode: standalone)` rule MUST NOT match
- **AND** the `.phone` MUST render at its original 360×720 dimensions with the dark `box-shadow` ring and 32px `border-radius` intact
- **AND** the desktop walkthrough framing MUST be preserved
