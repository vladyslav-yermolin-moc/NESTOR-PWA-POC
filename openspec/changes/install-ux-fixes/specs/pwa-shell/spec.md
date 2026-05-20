# Delta for PWA Shell

This delta amends two existing requirements from `openspec/specs/pwa-shell/spec.md` (introduced by the `pwa-shell` change) and adds two net-new requirements covering the safe-area layout fix and the install-guidance banner.

## MODIFIED Requirements

### Requirement: PWA-Specific Meta Tags for iOS

The `<head>` of `index.html` MUST include the iOS-specific PWA meta tags so that "Add to Home Screen" on iOS Safari 17+ produces a standalone-mode launch (no Safari chrome) with the correct title, icon, and safe-area-respecting layout. The required tags are:

- `<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">`  *(MODIFIED — the `viewport-fit=cover` opt-in is now required so iOS returns non-zero `env(safe-area-inset-*)` values in standalone mode)*
- `<meta name="apple-mobile-web-app-capable" content="yes">`
- `<meta name="apple-mobile-web-app-status-bar-style" content="default">`
- `<meta name="apple-mobile-web-app-title" content="NestorAI">`
- `<link rel="apple-touch-icon" href="icons/icon-192.png">`

*(Previously: viewport tag was `width=device-width, initial-scale=1.0` without `viewport-fit=cover`, which caused safe-area variables to resolve as 0 and produced layout gaps and clipping on devices with a notch / home indicator.)*

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

### Requirement: Prototype Backward Compatibility

The prototype HTML's existing markup, screen flow, and inline JavaScript MUST NOT be modified by PWA-related changes. The permitted edits to the prototype source file (`PWA/2026-05-19-mvp-showing.html`) are limited to one `@media (display-mode: standalone), (display-mode: fullscreen)` CSS block appended inside the existing `<style>` element. The permitted additions in `PWA/index.html` beyond the prototype source are: (a) PWA `<head>` tags (including the viewport opt-in), (b) the SW-registration `<script>` near `</body>`, **(c) one install-guidance block (an inline `<style>`, a `<div id="install-banner">`, and an inline `<script>`) appended immediately after the SW-registration `<script>`.** *(MODIFIED — adds (c) to the previous allowlist.)*

Opening either file directly via `file://` MUST render the prototype exactly as the original (un-edited) file does — the `@media (display-mode: standalone)` block does not activate in browser-tab mode, and the install banner self-hides on detection of an unsupported context.

*(Previously: only (a) PWA head tags and (b) the SW-registration script were permitted additions to `index.html`.)*

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

## ADDED Requirements

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

### Requirement: Platform-Aware Install Guidance Banner

When the page is rendered in a browser tab (i.e., NOT in `display-mode: standalone` / `fullscreen`) and the user has not previously dismissed the banner, `index.html` MUST display a small dismissable banner that guides the user toward installing the PWA in a platform-appropriate way. The banner MUST be self-contained (HTML + inline CSS + inline JS appended to `index.html`), require no external dependencies, and never appear inside the installed PWA.

On iOS Safari: the banner copy MUST include the iOS Share icon and the literal instruction "Tap Share → Add to Home Screen" (or copy of equivalent meaning).

On Android (or any browser that fires `beforeinstallprompt`): the banner MUST capture `beforeinstallprompt`, suppress the browser's native prompt with `event.preventDefault()`, and render an in-banner "Install" button that calls `event.prompt()` when tapped.

#### Scenario: Banner appears in browser view on iPhone Safari

- **GIVEN** a first-time visitor opens the deployed URL in mobile Safari on iOS 17+
- **AND** `localStorage['nestor-install-dismissed']` is not set
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

#### Scenario: Banner dismissal persists across reloads

- **GIVEN** the banner is visible
- **WHEN** the user taps the × close button
- **THEN** `localStorage['nestor-install-dismissed']` MUST be set to `'1'`
- **AND** the banner MUST be hidden immediately
- **AND** on subsequent page loads the banner MUST NOT re-appear until the localStorage flag is cleared

#### Scenario: Android Chrome shows in-banner Install button

- **GIVEN** the page loads in Chrome 120+ on Android
- **AND** the page meets Chrome's installability heuristics (manifest + active SW + HTTPS)
- **WHEN** Chrome fires `beforeinstallprompt`
- **THEN** the script MUST call `event.preventDefault()` to suppress the browser's native prompt
- **AND** the captured event MUST be stored
- **AND** the banner MUST appear with an "Install" button
- **AND** when the user taps "Install", `event.prompt()` MUST be called
- **AND** on user acceptance, the banner MUST hide and `localStorage['nestor-install-dismissed']` MUST be set

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
