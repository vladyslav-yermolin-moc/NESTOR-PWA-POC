# Change: install-ux-fixes

## Why

On-device testing of the deployed PWA (`https://vladyslav-yermolin-moc.github.io/NESTOR-PWA-POC/`, installed on iPhone Safari 17 + observed in Chrome / Safari browser tabs on iPhone 16 Pro on 2026-05-20) surfaced four issues that block the POC's pass criterion ("feels like a native app on Nora's iPhone — and looks right when she first opens the URL"). Screenshots are in `docs/extenal_docs/`:

1. **Standalone-mode layout breaks under iOS chrome.** A large white gap appears between the iOS status bar and the page header; the bottom disclaimer footer is cropped mid-sentence ("...Buyer responsible for" — missing "final decisions."). Root cause: standalone-mode CSS (introduced in `pwa-shell`) resizes `.phone` to fill the viewport but does not opt into iOS safe-area handling (no `viewport-fit=cover`), and the desktop `display: flex; align-items: center` rule on `body` is not reset for standalone mode.

2. **No install guidance for visitors.** Without `beforeinstallprompt` on iOS Safari, Nora has no on-screen affordance telling her how to install — the install gesture is buried in the Share menu. Best-practice PWAs ship a small dismissable banner with platform-specific instructions.

3. **Mobile browser view has side-margin overflow and lost phone-frame.** In Chrome and Safari browser tabs on iPhone 16 Pro, the page renders without the desktop phone-frame and with content shifted off the left edge (avatar/text cropped at the viewport left). Root causes: (a) the `.phone`'s decorative `box-shadow: 0 30px 60px -15px ...` projects ~45px of blur beyond each side of the element, which on a 393pt-wide viewport pushes the page's effective scroll width past the visible area; (b) the desktop "phone preview" framing was never intended to render on a real mobile viewport, but the current CSS keeps trying to draw it at 360px-fixed inside a smaller usable area; (c) no `overflow-x: hidden` safety net on `body`. This is the **first thing Nora sees** when opening the URL — it cannot look broken before she even installs.

4. **Schedule-a-Showing date / time inputs look broken on iOS.** On iPhone, `.showing-date-input` (type=date) appears as a thin near-empty pill — iOS Safari ignores the `placeholder` attribute on date inputs, so there is no visible affordance until the user taps. The `font-size: 13px` triggers iOS Safari's auto-zoom on focus (zoom in, can't easily zoom out). The neighbouring Hour / AM-PM selects have the same problem. Net effect: the showing-scheduling modal — a core Nora-facing flow — looks unfinished.

This change addresses all four. The fixes are additive (per ADR-001's "PWA features are additive" constraint) and do not modify the prototype's screen flow, only specific CSS rules and the bottom-of-body PWA additions.

## What Changes

**Layout fix (Issue 1):**

- **Modify `index.html` `<head>`**: update the existing viewport meta tag from `width=device-width, initial-scale=1.0` → `width=device-width, initial-scale=1.0, viewport-fit=cover`. This is the iOS opt-in for `env(safe-area-inset-*)` to return non-zero values inside standalone PWAs on devices with a notch or home indicator.
- **Modify `2026-05-19-mvp-showing.html`** (the prototype source): extend the existing `@media (display-mode: standalone), (display-mode: fullscreen)` block to:
  - Reset `html, body { display: block; }` so the desktop flex-centering does not affect the full-viewport layout.
  - Switch `.phone { height: 100% }` → `height: 100vh; height: 100dvh;` (dynamic viewport height, which accounts for the iOS bottom toolbar correctly; `100vh` is the fallback for browsers without `dvh` support).
  - Add `padding-top/bottom/left/right: env(safe-area-inset-*)` to `.phone` so the inner header/content/footer all sit inside the safe area.
  - Add `box-sizing: border-box` to keep the dimensions correct under the new padding.

**Install guidance (Issue 2):**

- **Modify `index.html`**: append a third additive block just before `</body>` (alongside the existing SW-registration `<script>`):
  - A `<div id="install-banner" hidden>` that lives as a fixed-position bottom-of-screen banner with the inline `<style>` and `<script>` to drive it.
  - The `<script>` detects: (a) already-installed → return immediately; (b) iOS Safari (UA check) → render iOS Share-icon + "Tap Share → Add to Home Screen"; (c) Android Chrome → capture `beforeinstallprompt`, render an in-banner "Install" button that calls `prompt()`; (d) desktop or other → no banner.
  - Dismiss button (×) sets `sessionStorage['nestor-install-dismissed'] = '1'` so the banner stays hidden for the rest of the browser session (closing the tab clears it; the next session re-shows it).
  - Hidden by an inner `@media (display-mode: standalone), (display-mode: fullscreen) { display: none !important; }` so it never shows in the installed app, only in the browser tab view.

**Mobile browser-mode layout (Issue 3):**

- **Modify `2026-05-19-mvp-showing.html`** (the prototype source): extend the existing `@media (display-mode: standalone), (display-mode: fullscreen)` trigger list to also include `(max-width: 480px)`. When any of the three matches, the fullscreen-fill layout applies — i.e., the phone-frame chrome (`box-shadow`, `border-radius`, 360×720 fixed dimensions) disappears and `.phone` fills the viewport with safe-area padding. On desktop (> 480px in a browser tab), the phone-preview framing is preserved.
- **Modify `2026-05-19-mvp-showing.html`** `html, body`: add `overflow-x: hidden` as a safety net against any unintended horizontal overflow (covers the box-shadow blur and any future wide content). This is a global rule (not gated by media query) since horizontal scroll is undesirable in every mode of this prototype.

This revises a design decision from the original `pwa-shell` change (which deliberately kept the phone-frame in mobile-Safari browser view to preserve "visual continuity with the desktop walkthroughs"). Real-device feedback makes the continuity case weaker than the broken-first-impression case.

**Schedule-a-Showing date / time inputs (Issue 4):**

- **Modify `2026-05-19-mvp-showing.html`** CSS — three selectors:
  - `.showing-date-input`: `font-size: 13px` → `16px` (the iOS Safari zoom-on-focus threshold), `padding: 10px 38px 10px 12px` → `14px 44px 14px 14px`, add `min-height: 48px` (Apple HIG minimum touch target).
  - `.showing-time-select`: same `font-size`, `padding`, and `min-height` updates.
  - `.showing-ampm-select`: same updates plus widen `width: 90px` → `width: 110px` to fit the new font size.
  - `.showing-date-icon`: `font-size: 15px` → `20px`, `right: 11px` → `right: 14px` so the icon scales with the larger field.

The iOS date picker still opens natively on tap — these changes only resize the visible field so it reads as a real input affordance rather than a thin empty pill.

No prototype markup, screen JS, or screen flow is modified.

## Scope

### In Scope

- Adding `viewport-fit=cover` to the viewport meta tag in `index.html`.
- Extending the existing standalone-mode `@media` block (in the source prototype `2026-05-19-mvp-showing.html`) with safe-area padding, `dvh`/`vh` height, and body-flex reset. The same edit flows into `index.html` via the standard prototype → index copy.
- Extending the `@media` trigger to include `(max-width: 480px)` so mobile browser tabs (Chrome / Safari on real phones) also get the fullscreen-fill layout, eliminating the phone-frame box-shadow that caused horizontal overflow on iPhone 16 Pro.
- Adding `overflow-x: hidden` on `html, body` as a global horizontal-scroll safety net.
- Adding an iOS / Android install-guidance banner (HTML + inline CSS + inline JS) to `index.html` only, between the prototype content and `</body>`.
- sessionStorage-based dismissal memory so the banner is shown once per browser tab session (re-appears when the user closes the tab and re-opens the URL — suitable for the POC where Nora may want to see the banner again across visits).
- Hidden-in-standalone behavior so the banner never appears inside the installed PWA.
- Enlarging the showing-scheduling modal's date input + time / AM-PM selects to iOS-friendly sizing (16px font, 48px min-height, larger calendar icon) so the inputs read as real affordances on iPhone.

### Out of Scope

- Re-architecting the prototype to use modern layout primitives (e.g., grid, container queries). The fix uses minimal additive CSS only.
- iOS splash screen PNGs (the `apple-touch-startup-image` matrix) — still deferred per ADR-001/006.
- Push notifications, Web Share Target, background sync.
- A11y audit of the existing prototype content (separate concern; not regressed by this change).
- Internationalization of the banner copy. POC ships English-only.
- Telemetry for banner impressions/dismissals — no telemetry pipeline exists in the POC.
- Custom JS date-picker replacement. The native iOS date-picker (opened by tapping the resized field) is kept as-is.
- Pseudo-placeholder text inside the empty date input (iOS doesn't honour `placeholder` on `type=date`). If "Select a date" copy is later requested, that becomes a separate small change.

## Impact

**Affected files** (all in `PWA/` repo root):

- `index.html` — **modified**: one viewport meta tag value change (~1 line); one inline `<style>` + `<div>` + `<script>` block appended near the existing SW-registration script (~80–110 lines combined). Plus the propagated CSS changes from the source prototype copy.
- `2026-05-19-mvp-showing.html` — **modified**:
  - `html, body { overflow-x: hidden }` added (global, ~1 line).
  - The existing `@media (display-mode: standalone), (display-mode: fullscreen)` block gains a third trigger `(max-width: 480px)` and is extended with safe-area padding, `dvh` fallback, and body-flex reset (~7 lines added inside the existing block; no new selectors).
  - `.showing-date-input`, `.showing-time-select`, `.showing-ampm-select`, `.showing-date-icon` font-size / padding / min-height / icon-size updates (~10 lines of in-place value changes).

**Third-party CDN dependencies introduced:** none. The banner uses an inline SVG for the iOS share icon and pure CSS for styling.

**No DB migration required** (no database — ADR-003).

**Risks:**

- *Banner shows under cached/stale state.* If a visitor previously dismissed the banner under the old (broken-layout) deploy, they'll still see the dismissal flag from before. Acceptable for the POC — fresh visitors test the new banner; previously dismissed users can clear site data to re-test. Documented in tasks/smoke-test.
- *`100dvh` not supported in very old browsers.* All targeted browsers (iOS Safari 17+, Chrome 120+) support `dvh`. Fallback `height: 100vh` keeps older browsers functional; on iOS Safari < 16 the bottom may still get clipped, but those are out of our targeted matrix.
- *Banner contains text/markup not in the prototype source.* This is the third PWA-specific addition to `index.html`. The canonical spec `openspec/specs/pwa-shell/spec.md` constrains additions to two categories (head tags + SW-reg script). This change extends that allowlist in a spec delta — captured in `specs/pwa-shell/spec.md` as a MODIFIED requirement.
- *Dismissal is sessionStorage-scoped, not permanent.* Closing the tab/window clears the flag; the next session re-shows the banner. This was chosen deliberately for the POC so Nora (a single key user) can re-see the banner without manually clearing site data. For production a TTL or `localStorage` swap could be considered.

- *Desktop browser walkthroughs lose the phone-frame at narrow viewports.* The new `(max-width: 480px)` trigger fullscreen-fills any viewport ≤ 480px. A desktop window dragged below 480px wide will lose the phone preview. Acceptable — desktop walkthroughs happen on full-width windows; the breakpoint protects the on-device case where it actually matters.

- *Date-input visual is larger but still uses the native iOS picker.* Tapping the resized field still opens iOS's bottom-sheet date picker — the resize only addresses the visible affordance. If Nora specifically wants a custom in-modal calendar, that's a separate change.

**Team / process impact:**

- Ivan (QA) re-runs the iPhone smoke test on the updated deploy. Expected to take ~10 minutes.
- Vlad re-deploys via the `actions/deploy-pages` workflow (push to main is enough; no infra changes).
- Nora-share gate (smoke-test before share) re-applies — the URL is not re-shared until the iPhone smoke test passes against the new build.
