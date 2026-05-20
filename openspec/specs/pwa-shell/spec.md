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
---

### Requirement: PWA-Specific Meta Tags for iOS

The `<head>` of `index.html` MUST include the iOS-specific PWA meta tags so that "Add to Home Screen" on iOS Safari 17+ produces a standalone-mode launch (no Safari chrome) with the correct title, icon, and safe-area-respecting layout. The required tags are:

- `<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">` - `<meta name="apple-mobile-web-app-capable" content="yes">`
- `<meta name="apple-mobile-web-app-status-bar-style" content="default">`
- `<meta name="apple-mobile-web-app-title" content="NestorAI">`
- `<link rel="apple-touch-icon" href="icons/icon-192.png">`

#### Scenario: Viewport tag opts into safe-area handling

- **GIVEN** the deployed `index.html` is loaded on an iPhone with a notch or home indicator
- **WHEN** the browser parses the `<head>`
- **THEN** the `viewport` meta tag MUST include `viewport-fit=cover`
- **AND** `env(safe-area-inset-top)`, `env(safe-area-inset-bottom)`, `env(safe-area-inset-left)`, and `env(safe-area-inset-right)` MUST resolve to the device's actual inset values (non-zero on devices with chrome)

#### Scenario: iOS launches the installed app in standalone mode

- **GIVEN** Nora has added the POC to her iPhone home screen via Share → Add to Home Screen
- **WHEN** Nora taps the home-screen icon
- **THEN** the app MUST launch in a standalone window
- **AND** the Safari address bar MUST NOT be visible
- **AND** the home-screen icon label MUST read "NestorAI"
- **AND** the home-screen icon visual MUST be `icons/icon-192.png` (auto-scaled by iOS)

---
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
---

### Requirement: Prototype Backward Compatibility

The prototype HTML's existing markup, screen flow, and inline JavaScript MUST NOT be modified by PWA-related changes. The permitted edits to the prototype source file (`PWA/2026-05-19-mvp-showing.html`) are limited to one `@media (display-mode: standalone), (display-mode: fullscreen)` CSS block appended inside the existing `<style>` element. The permitted additions in `PWA/index.html` beyond the prototype source are: (a) PWA `<head>` tags (including the viewport opt-in), (b) the SW-registration `<script>` near `</body>`, **(c) one install-guidance block (an inline `<style>`, a `<div id="install-banner">`, and an inline `<script>`) appended immediately after the SW-registration `<script>`.** Opening either file directly via `file://` MUST render the prototype exactly as the original (un-edited) file does — the `@media (display-mode: standalone)` block does not activate in browser-tab mode, and the install banner self-hides on detection of an unsupported context.

#### Scenario: file:// opening renders prototype unchanged

- **GIVEN** a developer opens `PWA/index.html` via `file:///.../PWA/index.html`
- **WHEN** the page loads
- **THEN** all original prototype screens MUST render with the desktop phone-frame chrome intact (the `@media (display-mode: standalone)` block MUST NOT activate)
- **AND** the SW registration error MUST be silently caught
- **AND** the install banner MUST NOT appear (either because the platform check returns desktop, or because no `beforeinstallprompt` ever fires)

#### Scenario: Prototype diff vs index.html shows only the three permitted additions

- **GIVEN** a diff of `PWA/index.html` against `PWA/2026-05-19-mvp-showing.html`
- **WHEN** the diff is reviewed
- **THEN** the only changes MUST be: (a) additions inside `<head>` (manifest link, theme-color meta, four iOS meta/link tags, and the viewport-fit=cover modification), (b) a `<script>` block registering the service worker just before `</body>`, (c) a contiguous install-banner block (style + div + script) appended after (b)
- **AND** no existing `.screen`, `.phone`, or prototype-content CSS rule MUST be altered
- **AND** no original `<script>` or inline JS MUST be altered or relocated
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
---

### Requirement: Safe-Area-Respecting Standalone Layout

In standalone or fullscreen display mode (i.e., when launched from the home screen on iOS Safari 17+ or Android Chrome 120+), the `.phone` element MUST inset its content from each edge of the viewport by exactly the corresponding `env(safe-area-inset-*)` value. This ensures the prototype's header, scrollable content, and footer all sit within the device's safe area — clear of the iOS status bar, Dynamic Island / notch, and home indicator.

The `.phone` element MUST also use dynamic viewport height (`100dvh`) so that the layout adapts to runtime changes in browser chrome (URL bar shrink/expand). When the browser does not support `dvh`, the layout MUST fall back to `100vh` without breaking.

#### Scenario: Header sits below iOS status bar on iPhone with notch

- **GIVEN** Nora has installed the PWA on an iPhone with a notch (iPhone 14 / 15 / 16 series)
- **WHEN** she launches the app from the home screen
- **THEN** the `.phone-header` element's visual top edge MUST sit at or below the status bar / Dynamic Island
- **AND** no white gap larger than `env(safe-area-inset-top)` MUST appear between the status bar and the header
- **AND** the iOS status bar (time / signal / battery) MUST remain readable

#### Scenario: Footer disclaimer fully visible above the home indicator

- **GIVEN** the user has scrolled to the bottom of any prototype screen in installed mode
- **WHEN** the `.phone-footer` element is rendered
- **THEN** the entire disclaimer text MUST be visible, ending with "...Buyer responsible for final decisions."
- **AND** the footer MUST sit above the iOS home indicator (no overlap)
- **AND** the bottom safe-area padding MUST equal `env(safe-area-inset-bottom)`

#### Scenario: Dynamic viewport height accommodates browser chrome changes

- **GIVEN** the PWA is opened in mobile Safari browser tab mode (not yet installed)
- **WHEN** the user scrolls the page and the iOS bottom toolbar collapses
- **THEN** the `.phone` element height MUST track the new visible viewport (via `100dvh`)
- **AND** content MUST NOT be clipped at the new viewport bottom
- **AND** no scroll jitter MUST be introduced

#### Scenario: Body flex-centering does not affect standalone layout

- **GIVEN** the installed PWA renders in standalone mode
- **WHEN** the `@media (display-mode: standalone)` block applies
- **THEN** `html, body` MUST have `display: block` (not `display: flex`)
- **AND** the desktop centering rules (`align-items: center`, `justify-content: center`) MUST NOT apply
- **AND** the `.phone` element MUST fill the viewport edge-to-edge (modulo safe-area padding)

---
---

### Requirement: Platform-Aware Install Guidance Banner

When the page is rendered in a browser tab (i.e., NOT in `display-mode: standalone` / `fullscreen`) and the user has not previously dismissed the banner, `index.html` MUST display a small dismissable banner that guides the user toward installing the PWA in a platform-appropriate way. The banner MUST be self-contained (HTML + inline CSS + inline JS appended to `index.html`), require no external dependencies, and never appear inside the installed PWA.

On iOS Safari: the banner copy MUST include the iOS Share icon and the literal instruction "Tap Share → Add to Home Screen" (or copy of equivalent meaning).

On Android (or any browser that fires `beforeinstallprompt`): the banner MUST capture `beforeinstallprompt`, suppress the browser's native prompt with `event.preventDefault()`, and render an in-banner "Install" button that calls `event.prompt()` when tapped.

#### Scenario: Banner appears in browser view on iPhone Safari

- **GIVEN** a first-time visitor opens the deployed URL in mobile Safari on iOS 17+
- **AND** `sessionStorage['nestor-install-dismissed']` is not set
- **WHEN** the page finishes loading and the activation script runs
- **THEN** the banner MUST become visible within 3 seconds after `load`
- **AND** the banner MUST contain the iOS share icon (SVG inline)
- **AND** the banner copy MUST instruct the user to tap Share → Add to Home Screen

#### Scenario: Banner hidden inside installed PWA

- **GIVEN** the PWA is launched in standalone mode (installed app)
- **WHEN** the page loads
- **THEN** the `@media (display-mode: standalone), (display-mode: fullscreen)` rule that hides `.install-banner` MUST apply
- **AND** the banner element MUST NOT be visible to the user
- **AND** the `display` computed style of `.install-banner` MUST be `none`

#### Scenario: Banner dismissal persists within the browser session

- **GIVEN** the banner is visible
- **WHEN** the user taps the × close button
- **THEN** `sessionStorage['nestor-install-dismissed']` MUST be set to `'1'`
- **AND** the banner MUST be hidden immediately
- **AND** on subsequent page loads within the same browser session (tab/window not closed) the banner MUST NOT re-appear
- **AND** when the user closes the tab/window and re-opens the URL in a new session, the banner MUST re-appear (sessionStorage is cleared per-session, by design — see the dismissal-behavior decision in `design.md`)

#### Scenario: Android Chrome shows in-banner Install button

- **GIVEN** the page loads in Chrome 120+ on Android
- **AND** the page meets Chrome's installability heuristics (manifest + active SW + HTTPS)
- **WHEN** Chrome fires `beforeinstallprompt`
- **THEN** the script MUST call `event.preventDefault()` to suppress the browser's native prompt
- **AND** the captured event MUST be stored
- **AND** the banner MUST appear with an "Install" button
- **AND** when the user taps "Install", `event.prompt()` MUST be called
- **AND** on user acceptance, the banner MUST hide and `sessionStorage['nestor-install-dismissed']` MUST be set

#### Scenario: Desktop browser shows no banner

- **GIVEN** the page loads in a desktop browser (no iOS UA, no `beforeinstallprompt` fire)
- **WHEN** the activation script completes initialization
- **THEN** the banner MUST remain hidden
- **AND** no platform-specific content MUST be injected into the banner DOM
- **AND** no user-facing nag MUST appear

#### Scenario: Already-installed user does not see the banner

- **GIVEN** the user previously installed the PWA and now revisits the URL in a browser tab while also having the installed copy
- **AND** the browser tab session reports `display-mode: standalone` is false
- **WHEN** the activation script runs
- **THEN** the script MUST check `navigator.standalone === true` (iOS-specific) and treat that as already-installed
- **AND** if true, return without showing the banner

---
---

### Requirement: Mobile-Viewport Fullscreen-Fill Trigger

Any viewport with width ≤ 480px MUST receive the fullscreen-fill layout (no `.phone` `box-shadow`, no `border-radius`, no fixed 360×720 dimensions, full-viewport `.phone` with safe-area padding), regardless of `display-mode`. This is added to the existing `display-mode: standalone` and `display-mode: fullscreen` triggers as a third condition (`(max-width: 480px)`).

In addition, `html, body` MUST set `overflow-x: hidden` as a global safety net against horizontal scroll from box-shadows, embedded media, or future wide content.

#### Scenario: Mobile Safari browser tab renders fullscreen-fill

- **GIVEN** Nora opens the deployed URL on an iPhone 16 Pro (393pt viewport) in mobile Safari (browser tab mode; not yet installed)
- **WHEN** the page renders
- **THEN** the `@media (..., max-width: 480px)` rule MUST match (viewport 393pt ≤ 480px)
- **AND** the `.phone` MUST render at viewport width with `border-radius: 0` and `box-shadow: none`
- **AND** no content MUST be cropped at the left or right viewport edge
- **AND** the page MUST NOT have any horizontal scroll

#### Scenario: Desktop browser at narrow viewport still gets fullscreen-fill

- **GIVEN** a developer drags a desktop browser window to ≤ 480px wide
- **WHEN** the page re-renders at the new size
- **THEN** the same fullscreen-fill rules MUST apply
- **AND** the desktop phone-frame preview MUST disappear

#### Scenario: Desktop browser at normal width preserves phone-frame preview

- **GIVEN** a desktop browser at ≥ 481px wide showing the URL in a browser tab
- **WHEN** the page renders
- **THEN** the `@media (..., max-width: 480px)` rule MUST NOT match (viewport > 480px)
- **AND** the `@media (display-mode: standalone)` rule MUST NOT match (browser tab)
- **AND** the `.phone` MUST render at 360×720 with its desktop `box-shadow` ring and `border-radius`
- **AND** the body MUST flex-center the `.phone` on the gradient background

#### Scenario: Horizontal-scroll safety net is active

- **GIVEN** any viewport and any display mode
- **WHEN** the page renders
- **THEN** `html, body` MUST have `overflow-x: hidden`
- **AND** no horizontal scroll bar MUST appear regardless of any descendant element's effective rendered width

---
---

### Requirement: Schedule-a-Showing Modal Inputs Sized for iOS

The Schedule-a-Showing modal's three input fields — `.showing-date-input` (date), `.showing-time-select` (Hour), `.showing-ampm-select` (AM/PM) — MUST be sized so that on iOS Safari they (a) do not trigger auto-zoom on focus, (b) read as real tap-able input affordances even when empty, (c) meet Apple HIG's 44pt minimum touch target. The native iOS date/select pickers continue to open on tap; only the resting visual is changed.

#### Scenario: Inputs do not trigger iOS auto-zoom on focus

- **GIVEN** the Schedule-a-Showing modal is open on iPhone Safari 17+
- **WHEN** the user taps `.showing-date-input`, `.showing-time-select`, or `.showing-ampm-select`
- **THEN** iOS Safari MUST NOT zoom the page (i.e., the `font-size` of each input MUST be ≥ 16px)
- **AND** the existing native picker (date picker / select wheel) MUST still open

#### Scenario: Inputs meet minimum touch target

- **GIVEN** the modal is open
- **WHEN** the three inputs render
- **THEN** each of `.showing-date-input`, `.showing-time-select`, `.showing-ampm-select` MUST have a computed height of at least 48px (via `min-height: 48px` plus padding)

#### Scenario: Date input affordance is legible even when empty

- **GIVEN** `.showing-date-input` has no value selected yet
- **WHEN** the modal renders
- **THEN** the calendar icon (`.showing-date-icon`) MUST be visible at the right edge of the input at ≥ 20px font size
- **AND** the input field MUST be tall enough (≥ 48px) and visually framed that a user immediately recognizes it as a tappable field, not an empty pill

#### Scenario: AM-PM select fits its content

- **GIVEN** the `.showing-ampm-select` displays "AM" or "PM" plus the dropdown caret
- **WHEN** the modal renders on iPhone
- **THEN** the select MUST be at least 110px wide
- **AND** the displayed text MUST NOT be visually truncated
