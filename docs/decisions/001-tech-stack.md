---
status: accepted
date: 2026-05-19
category: tech-stack
---

# ADR-001: Tech Stack — Vanilla HTML/CSS/JS + PWA Shell

## Context and Problem Statement

The Nestor PWA POC's sole purpose is to give the client (Nora) an installable PWA on her iPhone so she can feel the technology firsthand before committing to PWA as the delivery vehicle for the full Nestor product. The starting payload is the existing HTML prototype at `2026-05-19-mvp-showing.html` (1191 LOC). The decision is what runtime, language, and framework to use to wrap that prototype as an installable PWA.

A framework (Vite + React, Next.js, etc.) would introduce a build step, a dev server, transpilation, and a dependency tree that the POC does not need — the existing prototype is already vanilla HTML/CSS/JS and is the artifact Nora has been reacting to in walkthroughs. Re-platforming it onto a framework would burn time without changing what Nora feels on-device.

## Decision Drivers

- **Speed to demo.** Nora needs an installable URL as quickly as possible. Every minute spent on tooling is a minute not spent on getting an HTTPS URL on her home screen.
- **No throwaway code.** The POC validates the PWA install experience, not the framework choice. Code written here is not expected to ship to the full product.
- **Match the existing artifact.** The prototype is already vanilla HTML/CSS/JS. Keeping the stack the same removes the risk of regressing what Nora has already seen.
- **Service worker simplicity.** A plain static site is the simplest possible service-worker scope: one document, one cache namespace.

## Considered Options

1. **Vanilla HTML/CSS/JS + manifest.webmanifest + service-worker.js** — wrap the existing prototype directly.
2. **Vite 7 + React 19 + vite-plugin-pwa** — the stack used for the full product's PoV at `NEREES/frontend/`.
3. **Next.js 15 + next-pwa** — heavier, built-in routing and SSR if the POC scope ever grows.

## Decision Outcome

**Chosen option:** "Vanilla HTML/CSS/JS + manifest.webmanifest + service-worker.js", because the existing prototype already is that stack, and the POC's only purpose is to validate the install/offline experience — not to evaluate any framework. Re-platforming the prototype onto Vite or Next.js would add hours of setup with zero impact on what Nora feels on-device.

### Consequences

- Good, because the existing 1191-LOC prototype becomes the POC with near-zero rework.
- Good, because there is no build, no dependency install, no version drift — the repo is what gets served.
- Good, because the entire site is one HTML document plus a manifest and a service worker — easiest possible service-worker scope.
- Bad, because none of the POC code carries forward to the full product, which is being built on Vite + React per the parent project's CLAUDE.md.
- Neutral, because the lack of TypeScript means no compile-time type safety — but the JS surface area is tiny.

## Pros and Cons of the Options

### Vanilla HTML/CSS/JS + PWA shell

- Good, because matches the existing prototype exactly — no porting work.
- Good, because zero build/install/transpile overhead.
- Bad, because no shared code path with the eventual full-product stack.

### Vite + React + vite-plugin-pwa

- Good, because aligns with the full-product stack the team will build later.
- Good, because vite-plugin-pwa handles manifest + service worker generation.
- Bad, because requires re-implementing the prototype as React components — non-trivial time cost for a POC.

### Next.js + next-pwa

- Good, because built-in routing / SSR if the POC scope grows.
- Bad, because much heavier than the POC needs — significant setup, more moving parts.
- Bad, because the full product is on Vite per `../specs/001-search-browse/plan.md`, so Next.js here would not even align with the eventual stack.

## Constraints on Future Work

- All new code in this repo MUST be written as vanilla HTML, CSS, and JavaScript — no React, Vue, Svelte, or other framework dependencies.
- There MUST NOT be a bundler, transpiler, or build step (no Vite, Webpack, esbuild, TypeScript compiler) — files are served as-authored.
- The PWA layer (`manifest.webmanifest`, `service-worker.js`) MUST be additive — opening the prototype HTML directly (e.g., `file://`) without the service worker registered MUST still render the prototype correctly.
- Third-party dependencies (fonts, icons) MUST be loaded via CDN `<link>` / `<script>` tags. There MUST NOT be a `package.json` or `node_modules/` in this repo.
- JavaScript MUST target evergreen mobile browsers (iOS Safari 17+, Chrome 120+). No transpilation, no legacy browser fallbacks.

## Deferred Aspects

None — the decision is fully scoped to the POC. A separate ADR will be written when (or if) the POC graduates to a full product build, at which point the stack will likely move to Vite + React per `../specs/001-search-browse/plan.md`.
