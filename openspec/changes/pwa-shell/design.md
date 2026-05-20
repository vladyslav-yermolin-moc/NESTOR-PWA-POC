# Design: pwa-shell

## Technical Approach

Wrap the existing 1191-LOC prototype (`2026-05-19-mvp-showing.html`) in the minimum PWA shell that makes it installable on iPhone Safari 17+ and Android Chrome 120+. The PWA layer is purely additive — no prototype markup, styles, or scripts are modified.

Five new files are added to the `PWA/` repo root:

```
PWA/
├── index.html              ← prototype + <head> PWA tags + SW registration
├── manifest.webmanifest    ← web app manifest (name, icons, display, colors)
├── service-worker.js       ← precache + cache-first + version cleanup
├── icons/
│   ├── icon-192.png        ← 192×192 PNG of CSS logo on #1F2937
│   └── icon-512.png        ← 512×512 PNG of CSS logo on #1F2937
└── (no package.json, no node_modules — per ADR-001)
```

The flow from "Nora opens URL" to "icon on home screen" is the only happy path we care about. Everything else (push, background sync, splash PNGs, install-guidance overlay) is deferred per the proposal's Out of Scope section.

## Architecture Decisions

### Decision: Prototype is now local to `PWA/`; `index.html` is a copy of it

**Context:** The prototype was relocated from `../2026-05-19-mvp-showing.html` into `PWA/2026-05-19-mvp-showing.html` so the entire POC is self-contained for GitHub Pages deployment (per ADR-005). Two same-folder files (`2026-05-19-mvp-showing.html` as the source, `index.html` as the deployable entry point) preserve a clean history: the dated source file is the unedited-except-for-display-mode-CSS prototype; the `index.html` adds the PWA wiring on top.

**Decision:** Keep both files in `PWA/`. `2026-05-19-mvp-showing.html` is the source of truth for prototype content (and is the only file edited in this change to add the standalone-mode CSS block). `index.html` is a byte-identical copy plus the PWA additions (`<head>` tags + SW registration `<script>`).

**Rationale:** Symlinks were rejected up front because GitHub Pages does not follow them. A single-file approach (rename the prototype to `index.html`, edit in place) would also work and is slightly simpler — but keeping the dated filename as the unmodified source makes the diff for PWA additions trivial to review (`diff 2026-05-19-mvp-showing.html index.html` shows exactly the PWA additions, nothing else).

### Decision: Remove the desktop phone-frame chrome in standalone mode via `@media (display-mode: standalone)`

**Context:** The prototype's `.phone` element wraps content in a 360×720 white box with a dark `box-shadow` ring and a 32px `border-radius` — a desktop "phone preview" presentation. When the PWA is installed on Nora's actual iPhone and launched from the home screen, this renders as a small phone-shaped panel inside the real phone screen (phone-inside-a-phone). The desktop walkthrough use case still wants the frame; the on-device installed use case does not.

**Decision:** Append a single `@media (display-mode: standalone), (display-mode: fullscreen)` block to the prototype's existing `<style>` element. The block resets:

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

This makes the `.phone` viewport-filling and removes the ring/curve only when the page runs as an installed PWA. Desktop browser view (and `file://` open) renders unchanged.

**Rationale:** Two alternatives considered:

- A `(max-width: 480px)` media query — also fixes the issue on mobile Safari browser view, but loses the "phone preview" framing when the bare URL is opened on a phone before install. Rejected because Nora may visit the URL in mobile Safari first; keeping the desktop preview consistent there preserves visual continuity with the desktop walkthroughs.
- Add a top-level wrapper class toggled by JS — overkill for a CSS-only concern.

The chosen approach is the minimal, declarative, browser-native trigger. It honors ADR-001's "PWA features are additive" rule because the @media block is gated on a display mode that only activates in an installed PWA context — the `file://` and desktop browser experiences are unchanged.

### Decision: Service worker scope = site root (`./` or repo subpath)

**Context:** GitHub Pages serves the repo at `https://<org>.github.io/<repo>/`. Per ADR-005, all paths in `manifest.webmanifest` and the service worker `scope` must be relative or include the subpath. Per ADR-002, the service-worker scope must be the site root.

**Decision:** Place `service-worker.js` at the `PWA/` root (same directory as `index.html`). Register with `navigator.serviceWorker.register('service-worker.js')` (no leading slash). The manifest's `start_url` and `scope` are both `"."`. All icon `src` values use relative paths (`"icons/icon-192.png"`).

**Rationale:** Relative paths resolve correctly under both `localhost:8000/` and `<org>.github.io/<repo>/` without configuration changes. Placing the SW in a subdirectory like `js/` would narrow its `scope` to that subdirectory and break installability — the SW must be served from the same scope as the documents it caches.

### Decision: Cache-first for same-origin, network-first with cache fallback for cross-origin (Google Fonts)

**Context:** Per ADR-001, the prototype loads Google Fonts via CDN `<link>` tags. The service worker has to make a per-origin strategy choice — too aggressive a cache breaks font updates; too lax a cache breaks offline.

**Decision:** Two-strategy approach:
- **Same-origin** (the app shell — HTML, manifest, icons): **cache-first**. The shell rarely changes, and offline-first is the entire reason the SW exists.
- **Cross-origin** (Google Fonts CSS + WOFF2 files): **network-first with cache fallback**. Fresh fonts when online; cached fonts when offline. Acceptable because font CSS rarely changes in practice.

**Rationale:** Cache-first everywhere would risk stale fonts forever (the prototype's Inter font URL has versioned hashes, so this would be invisible until Google rotates a CDN host). Network-first everywhere defeats offline. The split matches the actual change frequency of each origin.

### Decision: Cache versioning via a single `CACHE_VERSION` constant

**Context:** When the prototype is edited, the SW must invalidate stale shell entries on next install.

**Decision:** Hardcode `const CACHE_VERSION = 'nestor-pwa-v1';` at the top of `service-worker.js`. The `activate` handler deletes any `caches.keys()` that don't match the current version. Bumping the version (v1 → v2) on the next shell change cleanly invalidates all stale entries.

**Rationale:** This is the standard service-worker versioning pattern. A more sophisticated approach (hash of file contents, build-time injection) would require a build step, which ADR-001 prohibits. Manual version bumping is acceptable for a POC that updates rarely.

### Decision: SW registration script inlined in `index.html`, guarded by feature detection

**Context:** The registration script needs to run after the SW file exists at a known path. It must not break the page if `serviceWorker` is unavailable (e.g., when opened via `file://` for local prototype review).

**Decision:** Append the following inline script just before `</body>` in `index.html`:

```html
<script>
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('service-worker.js').catch(() => {
        // Silent failure — file:// or other unsupported context. POC continues without offline support.
      });
    });
  }
</script>
```

**Rationale:** Inlining keeps the PWA layer a single-file diff against the prototype (no separate `app.js`). Feature detection ensures the prototype still works when opened directly without HTTPS, satisfying ADR-001's "additive, not load-bearing" constraint. The empty `.catch()` swallows the registration error rather than logging it, since the POC has no telemetry pipeline.

### Decision: Two icon sizes (192px + 512px) — skip the larger matrix

**Context:** A complete icon set spans many sizes (48, 72, 96, 144, 192, 256, 384, 512). Most are required only for specific platform manifests (Microsoft tile, Chrome OS, etc.).

**Decision:** Ship exactly two PNGs: `icon-192.png` (Android home screen, Chrome install card) and `icon-512.png` (Android splash, Chrome PWA install dialog). Reference both in `manifest.webmanifest`. Add a single `<link rel="apple-touch-icon" href="icons/icon-192.png">` for iOS home screen.

**Rationale:** 192 and 512 are the minimum to satisfy Chrome's PWA installability heuristic. Apple Touch Icon uses 192 (iOS auto-scales it). Larger matrix is wasted bytes for a POC. If Nora reports icon-quality issues, a follow-up change adds more sizes.

### Decision: Defer iOS splash-screen PNGs

**Context:** iOS Safari ignores `manifest.webmanifest`'s `background_color` for the launch splash. Branded splash on iOS requires a stack of `<link rel="apple-touch-startup-image" sizes="..." href="...">` PNGs at exact device pixel sizes (iPhone SE, 14, 14+, 15, 15 Pro, 15 Pro Max — at minimum 6 variants, each in portrait and landscape).

**Decision:** Skip splash PNGs entirely for this change. iOS will show a brief white flash on launch — accepted for the POC.

**Rationale:** The POC tests whether PWA install behavior is viable. A white-flash launch still demonstrates standalone-mode launch (no Safari chrome) — the most important visual cue. Generating 12+ splash variants is design work that should not block the install test. If Nora reacts negatively to the white flash, a follow-up change adds the PNG matrix.

## Data Flow / Component Interactions

### Install flow (iOS Safari)

```
Nora (Safari iOS)            GitHub Pages           index.html         service-worker.js
       │                          │                      │                     │
       ├── GET / ─────────────────▶                      │                     │
       │                          │                      │                     │
       │◀─── 200 + index.html ────┤                      │                     │
       │                          │                      │                     │
       │  parse <head>:           │                      │                     │
       │   - <link rel="manifest">                       │                     │
       │   - <meta theme-color>   │                      │                     │
       │   - <link apple-touch-icon>                     │                     │
       │   - register('service-worker.js')               │                     │
       │                          │                      │                     │
       ├── GET /service-worker.js ▶                      │                     │
       │◀───── 200 ───────────────┤                      │                     │
       │                          │                      │                     │
       │  install event fires ────────────────────────────────────────────────▶│
       │                          │                      │                     │
       │                          │              ◀── cache.addAll([            │
       │                          │                     'index.html',          │
       │                          │                     'manifest.webmanifest',│
       │                          │                     'icons/icon-192.png',  │
       │                          │                     'icons/icon-512.png']) │
       │                          │                      │                     │
       │  activate event ─────────────────────────────────────────────────────▶│
       │                          │                ◀──── delete stale caches ──┤
       │                          │                      │                     │
       │  Nora taps Share → "Add to Home Screen" → Confirm                    │
       │                          │                      │                     │
       │  Home screen icon appears (192 PNG, scaled by iOS)                   │
       │                          │                      │                     │
       │  Nora taps icon → standalone window opens → white flash → app shell  │
       │                          │                      │                     │
       │  Subsequent launches (online or offline):                            │
       │  fetch event ─────────────────────────────────────────────────────────▶│
       │                          │              ◀──── cache.match(req) ──────┤
       │                          │              (cache-first for same-origin)│
```

### Install flow (Android Chrome)

Same as iOS except Chrome auto-shows an install card when the SW + manifest are valid. Nora taps "Install" rather than navigating the Share menu. Splash uses the manifest's `background_color` + 512 icon — no white flash.

### Runtime fetch strategy

```
fetch(request)
   │
   ▼
┌──────────────────────────────────┐
│  Is request URL same-origin?     │
└──────────┬───────────────────────┘
           │
    ┌──────┴──────┐
    │             │
   YES           NO  (e.g., fonts.googleapis.com)
    │             │
    ▼             ▼
cache-first    network-first
    │             │
    ▼             ▼
hit cache?    fetch network
yes → return     │
no  → fetch ─────┴── on success: cache + return
                  on failure:  cache.match → return (or fail)
```

## File Changes

All paths relative to `PWA/`:

| Path | Status | Purpose |
|---|---|---|
| `2026-05-19-mvp-showing.html` | **modified** | One `@media (display-mode: standalone), (display-mode: fullscreen)` block appended inside the existing `<style>` element (~10 lines). Resets `.phone` to viewport-filling and sets `html, body` background to white in installed-PWA mode. No other changes. |
| `index.html` | **new** | Byte-identical copy of `PWA/2026-05-19-mvp-showing.html` (post-CSS-edit) with PWA `<head>` block added and SW registration `<script>` appended before `</body>`. |
| `manifest.webmanifest` | **new** | Web app manifest. Fields: `name: "NestorAI"`, `short_name: "NestorAI"`, `start_url: "."`, `scope: "."`, `display: "standalone"`, `background_color: "#1F2937"`, `theme_color: "#1F2937"`, `icons` array with two PNG entries. |
| `service-worker.js` | **new** | Three event handlers: `install` (precache app shell into `nestor-pwa-v1`), `activate` (delete non-current caches), `fetch` (cache-first for same-origin, network-first with cache fallback for cross-origin). |
| `icons/icon-192.png` | **new** | 192×192 raster of the prototype's CSS X-in-box logo. Dark-mode color: `#1F2937` background, white strokes. |
| `icons/icon-512.png` | **new** | 512×512 version of the same logo. Used by Android splash and Chrome install dialog. |

The prototype source file (`2026-05-19-mvp-showing.html`) is the only existing file modified — a single `@media (display-mode: standalone)` block is appended inside `<style>`. All other files are net-new. No deletions.

### ADR Compliance

This design honors all six foundational ADRs:

- **ADR-001 (Tech Stack):** Pure HTML/CSS/JS, no framework, no build step, no `package.json`. ✓
- **ADR-002 (Architecture):** Single `index.html`, root-scoped service worker, static deployment. ✓
- **ADR-003 (Data Storage):** No storage introduced. The SW caches static assets only; that is not data storage in the ADR's sense. ✓
- **ADR-004 (API/Auth):** No API calls, no auth. The only network requests are for static assets (same-origin + Google Fonts). ✓
- **ADR-005 (Deployment):** All paths relative; ready for GitHub Pages subpath serving. ✓
- **ADR-006 (Testing):** Manual smoke test on real iPhone + Android is the acceptance gate. ✓
