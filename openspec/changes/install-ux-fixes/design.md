# Design: install-ux-fixes

## Technical Approach

Two independent fixes, both delivered as additive edits with zero new dependencies:

1. **Safe-area-aware standalone layout.** Extend the existing standalone-mode `@media` block in `2026-05-19-mvp-showing.html` (which is then copied verbatim into `index.html` per the established pwa-shell pattern). Add `viewport-fit=cover` to the viewport meta tag in `index.html` only (the source prototype keeps its desktop viewport meta).

2. **Install-guidance banner.** Append a self-contained block to `index.html` just before `</body>`: one `<div>` (the banner DOM), one inline `<style>` (banner styling + standalone-mode hide), one inline `<script>` (platform detection + dismissal memory + Android `beforeinstallprompt` capture). No external files, no build step.

Both fixes honor ADR-001 (vanilla HTML/CSS/JS, no framework, no bundler), ADR-002 (single `index.html`, root-scoped), and ADR-004 (no network calls beyond CDN font assets).

## Architecture Decisions

### Decision: Two layout primitives — `viewport-fit=cover` + `env(safe-area-inset-*)`

**Context:** iOS Safari ignores manifest `background_color` for the launch splash and does NOT honor safe-area variables by default. Without the right meta tag, `env(safe-area-inset-top/bottom)` returns `0px` even on devices with a notch. The result is content rendered under the iOS status bar (top) and under the home indicator (bottom), exactly what the screenshots show.

**Decision:** Update the viewport meta tag to `width=device-width, initial-scale=1.0, viewport-fit=cover`. Then add `padding-top: env(safe-area-inset-top)`, `padding-bottom: env(safe-area-inset-bottom)`, plus `padding-left` and `padding-right` for completeness, to the `.phone` element inside the standalone-mode `@media` block. Use `box-sizing: border-box` so the padding does not push content outside the viewport.

**Rationale:** This is the documented Apple-blessed path (see Apple HIG → "Adapting to the Display Insets"). It is the minimum set of changes that makes the prototype's flex layout respect device chrome. Alternative approaches (CSS-driven full-screen with negative margins, JS-detecting safe areas) are heavier and yield identical visual results.

### Decision: `100dvh` for `.phone` height, with `100vh` fallback

**Context:** Inside an installed PWA on iOS, the viewport size reported by `100vh` is the *initial* viewport — it does NOT update when the system toolbar shrinks/grows during gesture interactions. The result on mobile Safari and standalone iOS is that `.phone { height: 100vh }` can be either too short (leaving a strip at the bottom) or too tall (forcing scroll).

**Decision:** Use both declarations in this order:

```css
.phone {
  height: 100vh;     /* fallback */
  height: 100dvh;    /* preferred; respects dynamic viewport */
}
```

Browsers that don't understand `dvh` silently drop the second declaration; supporting browsers use it.

**Rationale:** Two-line cost, no JS, no resize listeners. Targeted matrix (iOS Safari 17+, Chrome 120+) supports `dvh`; older browsers degrade gracefully.

### Decision: Reset `display: flex; align-items: center; justify-content: center` on body in standalone mode

**Context:** The prototype's desktop layout uses body-level flex-centering to position the `.phone` mockup in the middle of the viewport. In standalone mode, where `.phone` already fills the viewport, the centering creates compounded artifacts (small top gaps, occasional clipping) because `align-items: center` interacts unexpectedly with `height: 100%` flex items.

**Decision:** Inside the standalone-mode `@media` block, set `html, body { display: block; }` to fully reset the desktop flex behavior. Combined with `.phone { width: 100%; height: 100dvh }`, the phone fills the screen without alignment math.

**Rationale:** `display: block` is the cleanest reset — it eliminates the flex container entirely, removing both `align-items` and `justify-content` from the equation. Alternative approaches (overriding only `align-items: stretch`, or rebuilding the centering using grid) introduced more CSS and didn't simplify the mental model.

### Decision: Install banner as a single self-contained block at the end of `index.html`

**Context:** The canonical spec for `pwa-shell` currently allows exactly two additive blocks in `index.html` beyond the prototype source: (a) PWA `<head>` tags, (b) the SW-registration `<script>` near `</body>`. The install banner is a third addition. It contains DOM, CSS, and JS.

**Decision:** Add one block immediately after the existing SW-registration `<script>`. The block consists of:

1. A `<style>` element with banner-specific selectors prefixed `install-banner-*`.
2. A `<div id="install-banner" hidden>` containing the banner DOM (close button + content area).
3. A `<script>` IIFE containing the entire activation logic.

Keeping the block contiguous and after the SW-reg script makes it trivial to remove or update as a unit. The `install-banner-*` class prefix prevents collision with prototype styles. The block is appended to `index.html` only, never to the source prototype — so `file://` opens of the source still render the bare prototype (per ADR-001's additive-PWA constraint).

**Rationale:** A separate file (`install-banner.js`) was rejected because it would add an extra network request and require service-worker cache updates. Inlining keeps the entire PWA layer in three files (HTML, manifest, SW), matching ADR-001.

### Decision: Platform detection via UA + `beforeinstallprompt` capture

**Context:** iOS Safari does not fire `beforeinstallprompt` and exposes no install API. Chrome on Android fires `beforeinstallprompt` *if* the page meets installability heuristics (manifest + SW + HTTPS + first interaction). Desktop browsers vary.

**Decision:** The banner script uses two complementary signals:

```javascript
const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent) && !window.MSStream;
const isStandalone = window.matchMedia('(display-mode: standalone)').matches
                     || window.navigator.standalone === true; // iOS-specific fallback
```

- `isStandalone === true` → return immediately, never show banner.
- `isIOS && !isStandalone` → render Share-icon + "Tap Share → Add to Home Screen" message. No `beforeinstallprompt` to wait for.
- Otherwise (Android, desktop Chrome) → listen for `beforeinstallprompt`, capture and preventDefault it, render the banner with an "Install" button that calls `event.prompt()` when clicked.

**Rationale:** UA sniffing is normally a code smell but here it's matching a documented browser behavior gap that has no feature-detection alternative. The `navigator.standalone` fallback catches iOS Safari, which doesn't fully implement `display-mode` media queries in older versions.

### Decision: Add `(max-width: 480px)` to the fullscreen-fill `@media` trigger list, plus `overflow-x: hidden` on `html, body`

**Context:** On-device screenshots in `docs/extenal_docs/` from iPhone 16 Pro (Chrome and Safari browser tabs) show the page rendering with content cropped at the left edge and no visible phone-frame chrome. The `.phone`'s decorative `box-shadow: 0 30px 60px -15px ...` projects ~45px of blur on each side; on a 393pt-wide viewport this puts the effective rendered width past the visible area, and combined with the body's flex-centering produces a leftward shift and horizontal scroll. The previously-rejected `(max-width: 480px)` alternative (see `pwa-shell` ADR-002 design) was based on a hypothesis that "Nora may visit the URL in mobile Safari first; keeping the desktop preview consistent there preserves visual continuity." Real-device evidence overturns that hypothesis — the desktop preview does not render consistently on mobile and instead looks broken.

**Decision:** Extend the existing standalone-mode `@media` query trigger list from:

```css
@media (display-mode: standalone), (display-mode: fullscreen) {
```

to:

```css
@media (display-mode: standalone), (display-mode: fullscreen), (max-width: 480px) {
```

This means: any viewport ≤ 480px wide (covers all current iPhones in portrait, including iPhone 16 Pro Max at 430pt) renders the fullscreen-fill layout — no `box-shadow` ring, no `border-radius`, no fixed 360×720 dimensions, full-viewport `.phone` with safe-area padding.

Additionally, add `overflow-x: hidden` to `html, body` as a global rule (not gated by media query) so any future wide content can never produce horizontal scroll.

**Rationale:** The breakpoint is at 480px because:

- It's above the largest current iPhone portrait width (430pt on 16 Pro Max).
- It's below the smallest "tablet" portrait width (744pt on iPad mini), so iPads in portrait still get the desktop preview if they wanted it.
- It uses `max-width` (not `min-width`) so the rule is additive to the existing standalone trigger without inverting the logic — desktop browsers > 480px still get the original phone-frame preview.

The `overflow-x: hidden` is belt-and-suspenders against any horizontal overflow from box-shadows, embedded media, or future content. It is a one-line global rule with no observed downsides on the prototype's existing screens.

This decision **reverses** the previous `pwa-shell` design choice that explicitly rejected the `(max-width: 480px)` approach. The reversal is recorded here and propagated into the spec delta (the `Prototype Backward Compatibility` requirement is amended to reflect the broader media-query trigger).

### Decision: Enlarge `.showing-date-input`, `.showing-time-select`, `.showing-ampm-select` to iOS-friendly sizing

**Context:** Screenshot 4 (`docs/extenal_docs/photo_5220182249351879611_y.jpg`) shows the Schedule-a-Showing modal with three input fields: "Preferred Date" (`<input type="date">`), "Hour" select, "AM/PM" select. All three render as thin near-empty pills on iPhone:

- iOS Safari ignores `placeholder` on `<input type="date">` — the field looks empty until tapped (only a small calendar icon visible at the right).
- `font-size: 13px` is below iOS Safari's 16px zoom-on-focus threshold — tapping any of the three triggers an automatic page zoom that's awkward to undo.
- `padding: 10px ...` produces ~38px total height, below Apple HIG's 44pt minimum touch target.

**Decision:** Resize the three inputs (and matching icon) in place. Specifically:

- `font-size: 13px` → `16px` on all three inputs. Eliminates iOS auto-zoom.
- `padding: 10px ...` → `14px ...` on all three.
- Add `min-height: 48px` to all three. Above 44pt HIG minimum, comfortable thumb target.
- `.showing-date-icon`: `font-size: 15px` → `20px`, `right: 11px` → `right: 14px`. Scales icon with field.
- `.showing-ampm-select`: `width: 90px` → `width: 110px`. Fits the new 16px font with "AM"/"PM" + dropdown caret.

No structural HTML changes. The native iOS date / number pickers still open on tap; only the resting-state visual is improved.

**Rationale:** This is the smallest-possible change that fixes both the "looks broken" perception (Nora's first reaction to the showing-scheduling modal) and the iOS auto-zoom regression. Alternatives considered:

- *Custom JS date picker.* Heavier; introduces a state machine and visual design work. Rejected for POC scope.
- *JS-managed pseudo-placeholder span ("Select a date").* Adds DOM + JS for a one-line copy fix. Rejected as Out of Scope (proposal section); the resized field on its own makes the affordance legible enough.
- *Lower the `font-size` to 12px on desktop and 16px only on mobile via media query.* Adds complexity without obvious benefit; the new 16px works fine on desktop too.

The choice is to ship a minimal, in-place CSS resize that addresses the immediate Nora-facing issue and leaves space for a richer redesign later if scope expands.

### Decision: `localStorage` dismissal memory

**Context:** Once a user dismisses the banner, it should not re-appear on every navigation. The POC has no backend so server-side preference storage is unavailable.

**Decision:** Set `localStorage['nestor-install-dismissed'] = '1'` on dismissal. On page load, return early if that key is present. No expiry — dismissed forever (until the user clears site data).

**Rationale:** Simplest dismissal pattern that survives reloads and re-visits. Acceptable for a POC; for production a TTL or a "ask me later" path could be added. The dismissal applies per origin per browser profile, which is the right granularity for an install banner.

## Data Flow / Component Interactions

### Page-load sequence with the install banner

```
Page load (browser tab on iPhone Safari)
   │
   ▼
SW registration fires (existing) ──► SW activates, caches shell
   │
   ▼
Inline IIFE runs
   │
   ├── localStorage['nestor-install-dismissed'] === '1'? ──► return (no banner)
   │
   ├── matchMedia('(display-mode: standalone)') OR navigator.standalone? ──► return (already installed)
   │
   ├── isIOS?
   │      └── setTimeout(1500ms) ──► render iOS content, banner.hidden = false
   │
   └── isAndroid?
          └── addEventListener('beforeinstallprompt', e)
                  └── e.preventDefault() + store as deferredPrompt
                       + render Android content with "Install" button
                       + banner.hidden = false
                       + on button click: deferredPrompt.prompt()
                            └── on userChoice 'accepted' or 'dismissed':
                                 dismiss banner + localStorage flag
```

### Standalone-mode CSS resolution

```
Browser tab (display-mode: browser)
   ├── @media (display-mode: standalone) MISMATCH
   ├── .phone uses default rules: 360×720, dark frame, box-shadow
   ├── body flex-centers the .phone
   └── Banner visible (if not dismissed)

Installed PWA (display-mode: standalone)
   ├── @media (display-mode: standalone) MATCH
   ├── .phone reset: width: 100%; height: 100dvh; safe-area padding
   ├── body display: block (no flex centering)
   ├── viewport-fit=cover honored: env(safe-area-inset-*) return non-zero
   └── Banner hidden via inner @media rule
```

## File Changes

All paths relative to `PWA/`:

| Path | Status | Purpose |
|---|---|---|
| `2026-05-19-mvp-showing.html` | **modified** | (a) Add `overflow-x: hidden` to `html, body` (1 line, global). (b) Extend the existing `@media (display-mode: standalone), (display-mode: fullscreen)` block — add `(max-width: 480px)` to the trigger list, plus safe-area padding, dual-unit height, body flex reset inside (~7 lines added). (c) Resize `.showing-date-input`, `.showing-time-select`, `.showing-ampm-select`, `.showing-date-icon` to iOS-friendly sizing (~10 lines of in-place value changes). |
| `index.html` | **modified** | (a) Viewport meta tag value change: `width=device-width, initial-scale=1.0, viewport-fit=cover` (1 line). (b) All CSS changes from the source prototype propagate via the standard prototype → index copy. (c) New self-contained install-banner block (style + div + script) appended after the SW-registration script (~80–110 lines). |

No new files. No deletions. No new directories.

### ADR Compliance

- **ADR-001 (Tech Stack):** vanilla CSS + vanilla JS, inline, no build step, no npm dependency. ✓
- **ADR-002 (Architecture):** still one `index.html`, still root-scoped, no new routes. ✓
- **ADR-003 (Data Storage):** `localStorage` is permitted for transient UI state per ADR-003's deferred-aspect language — the dismissal flag qualifies. ✓
- **ADR-004 (API / Auth):** no API calls, no auth. ✓
- **ADR-005 (Deployment):** all paths still relative, no infra change. ✓
- **ADR-006 (Testing):** manual smoke test on real iPhone re-runs. ✓
- **ADR (`2026-05-20-service-worker-caching-strategy`):** banner adds no new network requests, so the cache strategy is unchanged. ✓

This change does not introduce any new substantive architectural decisions that warrant a fresh ADR. All choices documented above are local design decisions within the existing ADR envelope. The `adrs/` folder for this change will contain a single `none.md` marker.
