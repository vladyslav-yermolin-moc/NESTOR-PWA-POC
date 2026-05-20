# Tasks

## Phase 1: Extend standalone-mode CSS in prototype source

- [x] 1.1 Replace the existing `@media (display-mode: standalone), (display-mode: fullscreen)` block in `PWA/2026-05-19-mvp-showing.html` with the extended version: `html, body { display: block; }`, `.phone { height: 100vh; height: 100dvh; padding: env(safe-area-inset-*); box-sizing: border-box; }` — [plan §Phase 1] / [spec: Safe-Area-Respecting Standalone Layout → Body flex-centering does not affect standalone layout + Header sits below iOS status bar + Footer disclaimer fully visible]

## Phase 2: Update viewport meta tag in `index.html`

- [x] 2.1 Change `<meta name="viewport" content="width=device-width, initial-scale=1.0">` to include `viewport-fit=cover` in `PWA/index.html` (only; source prototype keeps its tag) — [plan §Phase 2] / [spec: PWA-Specific Meta Tags for iOS → Viewport tag opts into safe-area handling]

## Phase 3: Propagate Phase 1 CSS into `index.html`

- [x] 3.1 Replace the standalone-mode block in `PWA/index.html` with the same extended version added to the source in 1.1, so the deployed entry point picks up the safe-area fixes — [plan §Phase 3] / [spec: Prototype Backward Compatibility → Prototype diff vs index.html shows only the three permitted additions]

## Phase 4: Add install-guidance banner to `index.html`

- [x] 4.1 Append the `<style>` block defining `.install-banner` + `.install-banner-close` + `.install-banner-title` + `.install-banner-btn` + `.ios-share-icon` + inner `@media (display-mode: standalone)` hide — [plan §Phase 4] / [spec: Banner hidden inside installed PWA]
- [x] 4.2 Append the `<div id="install-banner" hidden>` containing close button and content area — [plan §Phase 4]
- [x] 4.3 Append the IIFE `<script>` implementing: early return on dismissal or standalone, UA-based iOS detection with Share-icon copy, Android `beforeinstallprompt` capture + Install button, sessionStorage dismissal (per-tab-session) — [plan §Phase 4] / [spec: Banner appears in browser view on iPhone Safari + Banner dismissal persists within the browser session + Android Chrome shows in-banner Install button + Desktop browser shows no banner + Already-installed user does not see the banner]

## Phase 5: Mobile-viewport fullscreen trigger + horizontal-scroll safety net

- [x] 5.1 Add `overflow-x: hidden` to the `html, body` rule in `PWA/2026-05-19-mvp-showing.html` — [plan §Phase 5] / [spec: Mobile-Viewport Fullscreen-Fill Trigger → Horizontal-scroll safety net is active]
- [x] 5.2 Extend the `@media` trigger list in the same file from `(display-mode: standalone), (display-mode: fullscreen)` to also include `(max-width: 480px)` — [plan §Phase 5] / [spec: Mobile-Viewport Fullscreen-Fill Trigger → Mobile Safari browser tab renders fullscreen-fill + Desktop browser at narrow viewport still gets fullscreen-fill]

## Phase 6: Resize Schedule-a-Showing modal inputs for iOS

- [x] 6.1 Update `.showing-date-input` (font-size 16px, padding 14px 44px 14px 14px, min-height 48px) and `.showing-date-icon` (font-size 20px, right 14px) — [plan §Phase 6] / [spec: Schedule-a-Showing Modal Inputs Sized for iOS → Inputs do not trigger iOS auto-zoom + Date input affordance is legible even when empty + Inputs meet minimum touch target]
- [x] 6.2 Update `.showing-time-select` and `.showing-ampm-select` (font-size 16px, padding 14px 12px, min-height 48px; ampm width 110px) — [plan §Phase 6] / [spec: AM-PM select fits its content + Inputs do not trigger iOS auto-zoom]

## Phase 7: Deploy + iPhone smoke test

- [x] 7.1 Commit + push to `main` to trigger the GitHub Actions Pages deploy (done 2026-05-20)
- [x] 7.2 iPhone smoke test on live URL: install flow works, banner renders correctly (after banner-bug fix + sessionStorage change), mobile margins OK, modal inputs sized correctly. Confirmed by Vlad 2026-05-20
- [x] 7.3 Confirmed ready to share with Nora — install-ux-fixes scope complete
