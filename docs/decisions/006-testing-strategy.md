---
status: accepted
date: 2026-05-19
category: testing
---

# ADR-006: Testing Strategy — Manual Smoke Test on Real Devices

## Context and Problem Statement

The POC validates whether a PWA installs and feels right on Nora's iPhone. The only behaviors that matter are: (a) the "Add to Home Screen" prompt appears and completes, (b) the branded splash screen renders on launch, (c) all prototype screens render and navigation works, (d) the site still works offline after first load. None of these are well-covered by typical JS unit tests — they are install-flow and device-rendering behaviors. The decision is what testing approach to commit to for the POC.

## Decision Drivers

- **The interesting behaviors are device-level.** Service worker registration, install prompt, splash screen, offline cache — these are browser and OS behaviors, not unit-test territory.
- **No automated tooling justifies its setup cost.** The POC is small enough that a 5-minute manual run on a real phone is faster and more credible than a Playwright suite.
- **The team has a QA (Ivan Pavlovskyi).** Manual smoke tests are a normal part of the team's workflow.
- **Lighthouse exists for PWA audits.** A one-time Lighthouse PWA audit (run from Chrome DevTools) covers the manifest and service-worker correctness without any CI investment.

## Considered Options

1. **Manual smoke test on iPhone (Safari) and Android (Chrome).**
2. **Vitest unit tests + Playwright E2E + Lighthouse CI.**
3. **Lighthouse PWA audit only.**
4. **Defer — decide later.**

## Decision Outcome

**Chosen option:** "Manual smoke test on iPhone (Safari) and Android (Chrome)", because the POC's value proposition is what Nora feels on her actual phone, and the only credible way to verify that is on real devices. A one-time Chrome DevTools Lighthouse PWA audit is recommended as a supplementary check but is not gated.

### Consequences

- Good, because tests exercise the exact behavior that matters (real install, real splash, real offline).
- Good, because no CI investment, no test-framework setup, no maintenance cost.
- Good, because Ivan can sign off in 5 minutes per release.
- Bad, because there is no regression safety net — a future edit could silently break the manifest without anyone noticing until the next manual run.
- Neutral, because the POC will be regenerated wholesale or retired if Nora gives the technology a no — extensive regression coverage would be wasted.

## Pros and Cons of the Options

### Manual smoke test on real devices (chosen)

- Good, because tests the actual behaviors that matter.
- Good, because zero setup cost.
- Bad, because no regression detection between manual runs.

### Vitest + Playwright + Lighthouse CI

- Good, because would automate everything.
- Bad, because requires Node + npm + a build step, conflicting with ADR-001.
- Bad, because Playwright cannot reliably test the iOS Safari install prompt — the most important behavior.

### Lighthouse PWA audit only

- Good, because catches manifest and service-worker config errors automatically.
- Bad, because Lighthouse does not test the iOS install prompt or splash-screen behavior on a real device.
- Verdict: useful as a supplement, not as the sole strategy.

### Defer

- Bad, because the smoke-test approach is already obviously right — deferring buys nothing.

## Constraints on Future Work

- Before sharing the GitHub Pages URL with Nora, the install flow MUST be smoke-tested on at least one real iPhone (Safari 17+) and one real Android device (Chrome 120+). Simulators and emulators MUST NOT be the sole verification.
- Each merged change to the POC MUST be re-smoke-tested on a real iPhone before re-sharing the URL with Nora.
- Smoke-test pass criteria MUST include: (1) "Add to Home Screen" succeeds, (2) launch from the home-screen icon shows the branded splash, (3) all prototype screens render and navigation works, (4) after first load, opening the installed app in airplane mode still renders the shell.
- Automated tests (Vitest, Playwright, Lighthouse CI) MAY be added if the POC scope grows beyond a clickable demo. That decision creates a new ADR.

## Deferred Aspects

Automated test coverage is deferred indefinitely. Trigger to revisit: the POC is promoted to a multi-build product that needs regression safety between releases.
