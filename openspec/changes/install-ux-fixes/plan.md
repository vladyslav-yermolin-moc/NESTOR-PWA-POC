# Implementation Plan: install-ux-fixes

## Overview

Two independent fixes layered onto the existing `PWA/` static site. Five edits to two files (`2026-05-19-mvp-showing.html` and `index.html`), one human-driven smoke test on real iPhone, and a re-deploy via the existing GitHub Actions workflow.

References: `proposal.md`, `design.md`, `specs/pwa-shell/spec.md`. Constraint envelope: `docs/decisions/001-006` and `docs/decisions/2026-05-20-service-worker-caching-strategy.md`. No new ADRs (see `adrs/none.md`).

No database migration. No build step. No new dependencies. No infra changes — the existing `.github/workflows/deploy.yml` re-runs on push to main.

## Implementation Phases

### Phase 1: Extend the standalone-mode CSS in the prototype source

**Changes:**

- Modify the existing `@media (display-mode: standalone), (display-mode: fullscreen)` block inside `PWA/2026-05-19-mvp-showing.html` to add safe-area padding, dual-unit height, body-flex reset, and `box-sizing`.

**Code sample:**

Existing block:

```css
@media (display-mode: standalone), (display-mode: fullscreen) {
  html, body { background: #FFFFFF; }
  .phone {
    width: 100%;
    height: 100%;
    border-radius: 0;
    box-shadow: none;
  }
}
```

Replace with:

```css
@media (display-mode: standalone), (display-mode: fullscreen) {
  html, body {
    background: #FFFFFF;
    display: block;                                /* reset desktop flex centering */
  }
  .phone {
    width: 100%;
    height: 100vh;                                 /* fallback */
    height: 100dvh;                                /* preferred — tracks dynamic viewport */
    border-radius: 0;
    box-shadow: none;
    padding-top: env(safe-area-inset-top);
    padding-bottom: env(safe-area-inset-bottom);
    padding-left: env(safe-area-inset-left);
    padding-right: env(safe-area-inset-right);
    box-sizing: border-box;
  }
}
```

**Acceptance Criteria:**

- `2026-05-19-mvp-showing.html` diff vs the upstream source shows the extended block (instead of the original 4-line variant).
- Opening the file via `file://` in a desktop browser still renders the desktop phone-frame chrome (the `@media` block does not match in `display-mode: browser`).
- Maps to spec scenarios: **"Safe-Area-Respecting Standalone Layout → Body flex-centering does not affect standalone layout"**, **"Header sits below iOS status bar on iPhone with notch"**, **"Footer disclaimer fully visible above the home indicator"**.

**Dependencies:** None.
**Parallel:** Yes — independent of Phase 2.

**Agent Routing:**

- `frontend-dev`: task 1.1 (SEQ within phase).

**Review Checkpoint:**

- Vlad confirms the diff: only the `@media` block is changed; all other CSS rules untouched.

---

### Phase 2: Update viewport meta tag in `index.html`

**Changes:**

- In `PWA/index.html`, change the viewport meta tag value from `width=device-width, initial-scale=1.0` to `width=device-width, initial-scale=1.0, viewport-fit=cover`. This is the iOS opt-in for safe-area-inset variables.

**Code sample:**

```html
<!-- Before -->
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<!-- After -->
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
```

The source prototype keeps its original viewport tag — the opt-in only matters when the page runs as an installed PWA, and the source HTML is not the deployed entry point.

**Acceptance Criteria:**

- `index.html` viewport meta tag contains `viewport-fit=cover` exactly once.
- `2026-05-19-mvp-showing.html` viewport tag remains unchanged.
- Maps to spec scenario: **"PWA-Specific Meta Tags for iOS → Viewport tag opts into safe-area handling"**.

**Dependencies:** None.
**Parallel:** Yes — independent of Phase 1.

**Agent Routing:**

- `frontend-dev`: task 2.1 (SEQ within phase).

**Review Checkpoint:**

- Code reviewer verifies the change is a one-attribute edit in the viewport meta tag, with no other changes around it.

---

### Phase 3: Re-seed `index.html` from the updated source + re-apply existing PWA additions

**Changes:**

- The cleanest way to ensure `index.html` includes the updated standalone-mode CSS from Phase 1 is to:
  1. Copy the updated `PWA/2026-05-19-mvp-showing.html` to `PWA/index.html`.
  2. Re-apply the viewport-fit=cover edit (Phase 2) on the new copy.
  3. Re-apply the PWA head tags (manifest link, theme-color, four apple-mobile-web-app-* tags, apple-touch-icon link) that the previous `pwa-shell` change introduced.
  4. Re-apply the SW-registration `<script>` before `</body>`.

Alternatively, edit `index.html` in place to update only the standalone-mode block — which is equivalent if done carefully. The choice is up to the implementer; the spec scenario for the index/source diff still holds either way.

**Code sample:**

In-place edit (recommended for safety — avoids re-introducing the head tags by hand):

Open `index.html`, find the existing standalone-mode block, replace it with the extended block from Phase 1. Then verify the viewport tag change from Phase 2 is in place.

**Acceptance Criteria:**

- `diff PWA/index.html PWA/2026-05-19-mvp-showing.html` shows exactly the three permitted addition categories (PWA head tags incl. viewport-fit modification, SW-registration script, install-banner block from Phase 4) and no other differences.
- Maps to spec scenario: **"Prototype Backward Compatibility → Prototype diff vs index.html shows only the three permitted additions"**.

**Dependencies:** Phase 1 and Phase 2 (provides the updated CSS and viewport tag).
**Parallel:** No.

**Agent Routing:**

- `frontend-dev`: task 3.1.

**Review Checkpoint:**

- Code reviewer runs the diff and confirms the only differences match the spec's permitted-additions list. Reviewer also confirms `file://` opening of `index.html` still renders the prototype correctly in browser mode.

---

### Phase 4: Add the install-guidance banner to `index.html`

**Changes:**

- Append three contiguous blocks to `PWA/index.html` immediately after the existing SW-registration `<script>` (still before `</body>`):
  - An inline `<style>` defining `.install-banner` and its children.
  - A `<div id="install-banner" hidden>` containing the close button and the content area.
  - An inline `<script>` IIFE implementing platform detection, dismissal memory, and `beforeinstallprompt` capture.

**Code sample:**

```html
<style>
  .install-banner {
    position: fixed;
    bottom: 16px;
    left: 16px;
    right: 16px;
    max-width: 380px;
    margin: 0 auto;
    background: #1F2937;
    color: #fff;
    border-radius: 16px;
    padding: 18px 20px;
    box-shadow: 0 10px 30px rgba(0,0,0,0.28);
    z-index: 1000;
    font-size: 14px;
    line-height: 1.45;
    font-family: 'Inter', system-ui, sans-serif;
  }
  .install-banner[hidden] { display: none; }
  .install-banner-close {
    position: absolute; top: 8px; right: 10px;
    background: transparent; border: none; color: #9CA3AF;
    font-size: 24px; line-height: 1; cursor: pointer;
    padding: 6px 10px;
  }
  .install-banner-close:hover, .install-banner-close:active { color: #fff; }
  .install-banner-title { font-weight: 700; font-size: 15px; margin: 0 0 2px 0; padding-right: 30px; }
  .install-banner-subtitle { font-size: 12px; color: #9CA3AF; margin: 0 0 14px 0; }
  .install-banner-steps { display: flex; flex-direction: column; gap: 9px; }
  .install-banner-step { display: flex; align-items: center; gap: 10px; font-size: 13px; }
  .install-banner-step-num {
    flex-shrink: 0; width: 20px; height: 20px;
    background: #374151; border-radius: 50%;
    display: inline-flex; align-items: center; justify-content: center;
    font-size: 11px; font-weight: 700; color: #fff;
  }
  .install-banner-step-text { flex: 1; }
  .ios-share-icon { display: inline-block; vertical-align: -4px; width: 16px; height: 16px; }
  .install-banner-btn {
    margin-top: 12px; width: 100%;
    background: #fff; color: #1F2937;
    border: none; padding: 11px 14px;
    border-radius: 10px; font-weight: 600;
    font-size: 14px; cursor: pointer;
    font-family: inherit;
  }
  .install-banner-btn:hover, .install-banner-btn:active { background: #F3F4F6; }
  @media (display-mode: standalone), (display-mode: fullscreen) {
    .install-banner { display: none !important; }
  }
</style>

<div id="install-banner" class="install-banner" hidden>
  <button class="install-banner-close" aria-label="Dismiss">×</button>
  <div class="install-banner-content"></div>
</div>

<script>
  (function () {
    var STORAGE_KEY = 'nestor-install-dismissed';
    if (sessionStorage.getItem(STORAGE_KEY) === '1') return;

    var isStandalone = window.matchMedia('(display-mode: standalone)').matches
                       || window.navigator.standalone === true;
    if (isStandalone) return;

    var ua = navigator.userAgent;
    var isIOS = /iPad|iPhone|iPod/.test(ua) && !window.MSStream;

    var banner = document.getElementById('install-banner');
    var content = banner.querySelector('.install-banner-content');
    var closeBtn = banner.querySelector('.install-banner-close');

    closeBtn.addEventListener('click', function () {
      banner.hidden = true;
      try { sessionStorage.setItem(STORAGE_KEY, '1'); } catch (_) {}
    });

    if (isIOS) {
      content.innerHTML =
        '<div class="install-banner-title">Install NestorAI on your phone</div>' +
        '<div class="install-banner-subtitle">Fullscreen, no browser bar</div>' +
        '<div class="install-banner-steps">' +
          '<div class="install-banner-step">' +
            '<span class="install-banner-step-num">1</span>' +
            '<span class="install-banner-step-text">Tap ' +
              '<svg class="ios-share-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">' +
                '<path d="M5 11v10h14V11"/><path d="M12 3v12"/><path d="M7 8l5-5 5 5"/>' +
              '</svg> in Safari\'s toolbar</span>' +
          '</div>' +
          '<div class="install-banner-step">' +
            '<span class="install-banner-step-num">2</span>' +
            '<span class="install-banner-step-text">Choose <strong>Add to Home Screen</strong></span>' +
          '</div>' +
          '<div class="install-banner-step">' +
            '<span class="install-banner-step-num">3</span>' +
            '<span class="install-banner-step-text">Tap <strong>Add</strong> (top right)</span>' +
          '</div>' +
        '</div>';
      setTimeout(function () { banner.hidden = false; }, 1500);
    } else {
      window.addEventListener('beforeinstallprompt', function (e) {
        e.preventDefault();
        var deferredPrompt = e;
        content.innerHTML =
          '<div class="install-banner-title">Install NestorAI</div>' +
          '<div class="install-banner-subtitle">Faster, fullscreen experience</div>' +
          '<button class="install-banner-btn" id="install-trigger">Install</button>';
        banner.hidden = false;
        document.getElementById('install-trigger').addEventListener('click', function () {
          if (!deferredPrompt) return;
          deferredPrompt.prompt();
          deferredPrompt.userChoice.then(function (choice) {
            if (choice.outcome === 'accepted') {
              banner.hidden = true;
              try { sessionStorage.setItem(STORAGE_KEY, '1'); } catch (_) {}
            }
            deferredPrompt = null;
          });
        });
      });
    }
  })();
</script>
```

**Acceptance Criteria:**

- The block is appended after the SW-registration `<script>`, before `</body>`.
- Opening `index.html` via `file://` does not show the banner (UA detection treats most desktop contexts as no-banner; iOS desktop simulation may show it, which is acceptable).
- In a real mobile Safari load, the banner appears after ~1.5 seconds with iOS-specific content.
- Tapping × hides the banner and sets `sessionStorage['nestor-install-dismissed']` to `'1'` (cleared automatically on tab/window close).
- On reload after dismissal, the banner does NOT re-appear.
- The `@media (display-mode: standalone)` rule inside the banner's `<style>` hides the banner inside the installed PWA.
- Maps to spec scenarios: **"Platform-Aware Install Guidance Banner → Banner appears in browser view on iPhone Safari"**, **"Banner hidden inside installed PWA"**, **"Banner dismissal persists across reloads"**, **"Android Chrome shows in-banner Install button"**, **"Desktop browser shows no banner"**, **"Already-installed user does not see the banner"**.

**Dependencies:** Phase 3 (block is appended to the same `index.html`).
**Parallel:** No.

**Agent Routing:**

- `frontend-dev`: tasks 4.1 (style), 4.2 (div), 4.3 (script) — SEQ within phase.

**Review Checkpoint:**

- Code reviewer verifies: (a) all three sub-blocks are inside `index.html` only, not in the source prototype; (b) selectors are prefixed `install-banner-*` to avoid collision; (c) the inner `@media (display-mode: standalone)` self-hide rule is present; (d) the IIFE returns early on standalone or prior dismissal; (e) the iOS detection is the documented regex; (f) the Android branch listens for `beforeinstallprompt` and uses `preventDefault()` correctly.

---

### Phase 5: Mobile-viewport fullscreen trigger + horizontal-scroll safety net

**Changes:**

- In `PWA/2026-05-19-mvp-showing.html`:
  1. Add `overflow-x: hidden` to the existing `html, body` rule (~1 line).
  2. Extend the `@media` trigger list (already extended in Phase 1) to add a third condition: `(max-width: 480px)`.

**Code sample:**

```css
/* Add to html, body rule near the top */
html, body {
  margin: 0; padding: 0; height: 100%;
  font-family: 'Inter', system-ui, sans-serif;
  color: #111827;
  background: radial-gradient(circle at 50% 50%, #fff, transparent 70%), linear-gradient(180deg, #F3F4F6, #E5E7EB);
  display: flex; align-items: center; justify-content: center;
  overflow-x: hidden;            /* NEW — global horizontal-scroll safety net */
}

/* Update the trigger list — was: standalone, fullscreen */
@media (display-mode: standalone), (display-mode: fullscreen), (max-width: 480px) {
  html, body { display: block; background: #FFFFFF; }
  .phone {
    width: 100%;
    height: 100vh;
    height: 100dvh;
    border-radius: 0;
    box-shadow: none;
    padding-top: env(safe-area-inset-top);
    padding-bottom: env(safe-area-inset-bottom);
    padding-left: env(safe-area-inset-left);
    padding-right: env(safe-area-inset-right);
    box-sizing: border-box;
  }
}
```

**Acceptance Criteria:**

- Mobile Safari + Chrome browser tab on iPhone 16 Pro show `.phone` filling the viewport with no horizontal overflow and no cropped content (validated against screenshots in `docs/extenal_docs/`).
- Desktop browser at full window width still shows the phone-frame preview centered on the gradient background.
- Computed style on `body` has `overflow-x: hidden` in every mode.
- Maps to spec scenarios: **"Mobile-Viewport Fullscreen-Fill Trigger → Mobile Safari browser tab renders fullscreen-fill"**, **"Desktop browser at narrow viewport still gets fullscreen-fill"**, **"Desktop browser at normal width preserves phone-frame preview"**, **"Horizontal-scroll safety net is active"**.

**Dependencies:** Phase 1 (the `@media` block exists). Re-apply the resulting CSS into `index.html` (Phase 3 already covers this, but it needs to be re-run if Phase 5 is added after Phase 3 — see Parallelization Notes).
**Parallel:** Yes — independent of Phase 4 (banner).

**Agent Routing:**

- `frontend-dev`: tasks 5.1 (global rule), 5.2 (extend trigger).

**Review Checkpoint:**

- Code reviewer confirms the `@media` query trigger list now has three conditions, and `overflow-x: hidden` is on `html, body`. No other CSS rules changed.

---

### Phase 6: Resize Schedule-a-Showing modal inputs for iOS

**Changes:**

- In `PWA/2026-05-19-mvp-showing.html`, modify four selectors in place:
  1. `.showing-date-input`
  2. `.showing-time-select`
  3. `.showing-ampm-select`
  4. `.showing-date-icon`

No new selectors, no DOM changes — only value updates inside the existing rules.

**Code sample:**

```css
.showing-date-input {
  width: 100%; border: 1px solid #D1D5DB; border-radius: 10px;
  padding: 14px 44px 14px 14px;       /* was: 10px 38px 10px 12px */
  font-size: 16px;                     /* was: 13px — avoids iOS auto-zoom */
  min-height: 48px;                    /* NEW — HIG touch target */
  color: #374151;
  font-family: inherit; outline: none; appearance: none; -webkit-appearance: none;
  background: #fff;
}

.showing-date-icon {
  position: absolute; right: 14px;     /* was: 11px */
  top: 50%; transform: translateY(-50%);
  color: #9CA3AF; pointer-events: none;
  font-size: 20px;                     /* was: 15px */
}

.showing-time-select {
  flex: 1; border: 1px solid #D1D5DB; border-radius: 10px;
  padding: 14px 12px;                  /* was: 10px 12px */
  font-size: 16px;                     /* was: 13px */
  min-height: 48px;                    /* NEW */
  color: #374151;
  font-family: inherit; outline: none; background: #fff;
  appearance: none; -webkit-appearance: none;
  background-image: url("data:image/svg+xml,...");  /* unchanged */
  background-repeat: no-repeat; background-position: right 10px center;
  padding-right: 30px;
}

.showing-ampm-select {
  width: 110px;                        /* was: 90px */
  flex-shrink: 0;
  border: 1px solid #D1D5DB; border-radius: 10px;
  padding: 14px 12px;                  /* was: 10px 12px */
  font-size: 16px;                     /* was: 13px */
  min-height: 48px;                    /* NEW */
  color: #374151;
  font-family: inherit; outline: none; background: #fff;
  appearance: none; -webkit-appearance: none;
  background-image: url("data:image/svg+xml,...");  /* unchanged */
  background-repeat: no-repeat; background-position: right 10px center;
  padding-right: 28px;
}
```

**Acceptance Criteria:**

- Computed `font-size` on all three input/select fields ≥ 16px (verified in DevTools or by tapping on iPhone and observing no auto-zoom).
- Computed `min-height` on all three ≥ 48px.
- `.showing-date-icon` is visibly larger than before and sits ~14px from the right edge of the input.
- The "AM"/"PM" select fits "AM" or "PM" plus the caret without truncation.
- Native iOS date / select pickers still open on tap.
- Maps to spec scenarios: **"Schedule-a-Showing Modal Inputs Sized for iOS → Inputs do not trigger iOS auto-zoom"**, **"Inputs meet minimum touch target"**, **"Date input affordance is legible even when empty"**, **"AM-PM select fits its content"**.

**Dependencies:** None for this phase's edits. Result propagates to `index.html` via Phase 3 (or via a re-run of the copy).
**Parallel:** Yes — independent of Phase 4 and Phase 5.

**Agent Routing:**

- `frontend-dev`: tasks 6.1 (date input + icon), 6.2 (time + ampm selects).

**Review Checkpoint:**

- Code reviewer confirms the four selectors changed in place, no new selectors introduced, no other rules touched. Visual smoke test: open Schedule-a-Showing modal in desktop browser — date and time fields now feel like normal-sized form fields, not pills.

---

### Phase 7: Re-run iPhone smoke test on the redeployed build

**Changes:**

- Push to `main` (triggers `actions/deploy-pages` workflow). Wait for green build. Open the Pages URL on Nora's iPhone (or Vlad's test iPhone). Re-run the smoke test.

**Code sample:**

Smoke-test checklist (record date, device, OS, browser, result):

```text
Pre-deploy:
  [ ] Local file://  open shows desktop phone-frame intact (no banner, no standalone CSS)
  [ ] Diff PWA/index.html vs PWA/2026-05-19-mvp-showing.html matches spec scenario

iPhone (Safari 17+) after deploy:
  [ ] Mobile Safari browser tab: .phone fills the viewport (NO phone-frame box-shadow / dark ring)
  [ ] No content cropped at the left or right edges; no horizontal scroll
  [ ] Install banner appears at the bottom after ~1.5s with iOS share-icon copy
  [ ] Tap × on banner → banner hides → reload → banner stays hidden
  [ ] Clear site data → reload → banner reappears
  [ ] Share → Add to Home Screen → confirm
  [ ] Tap home-screen icon → standalone launch:
        [ ] NO white gap above the header (header sits just below status bar)
        [ ] Footer disclaimer ends with "...final decisions." (NOT truncated)
        [ ] Content scrolls within .phone-content; no page-level scroll
        [ ] Install banner is NOT visible
  [ ] Airplane Mode → re-launch → shell still renders (offline still works)
  [ ] Tap through 3 prototype screens → all render with correct safe-area padding
  [ ] Open Schedule-a-Showing modal: date + time + AM/PM fields look like real-sized inputs (not thin pills)
  [ ] Tap any of the three fields → no page auto-zoom on focus; native picker opens normally

Mobile Chrome on iPhone 16 Pro:
  [ ] Browser tab: .phone fills viewport; no left/right cropping; no horizontal scroll
  [ ] Install banner with iOS share-icon copy (Chrome on iOS uses WebKit underneath, so iOS detection applies)

Android (Chrome 120+) — optional this round:
  [ ] Install banner shows Android-style "Install" button via beforeinstallprompt
  [ ] Tapping Install opens the Chrome native install prompt
  [ ] Install completes → home-screen icon → standalone launch with no layout gaps
```

**Acceptance Criteria:**

- All iPhone checks pass before the URL is re-shared with Nora.
- Android check is optional this round per the previous change's deferred Android testing.
- Maps to all spec scenarios end-to-end.

**Dependencies:** Phases 1–4 complete and pushed to `main`; deployment succeeds.
**Parallel:** No — verification is the last step.

**Agent Routing:**

- `tester`: task 5.1 (iPhone). Ivan or Vlad runs this.

**Review Checkpoint:**

- Vlad confirms all iPhone checkboxes pass. Any failure blocks share-with-Nora.

---

## Parallelization Notes

- **Wave 1 (parallel):** Phase 1 (standalone CSS extension), Phase 2 (viewport meta tag), Phase 5 (mobile-viewport trigger + overflow-x), Phase 6 (modal input resize). All four edit different sections of the source prototype (or `index.html` for Phase 2) and are independent.
- **Wave 2 (sequential, after Wave 1):** Phase 3 (propagate all source-prototype changes into `index.html` in one pass).
- **Wave 3 (sequential, after Wave 2):** Phase 4 (install-banner block in `index.html`).
- **Wave 4 (sequential, after Wave 3):** Phase 7 (deploy + smoke test).

**Database migration:** N/A.

**Rollback strategy:** Each phase touches plain static files. Revert with `git revert` of the deploy commit (or `git checkout HEAD~1 -- index.html 2026-05-19-mvp-showing.html`) and push. Pages will redeploy the previous build within a few minutes. No data loss, no migration roll-back, no infra changes.
