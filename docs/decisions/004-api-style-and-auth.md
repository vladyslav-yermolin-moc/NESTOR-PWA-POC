---
status: accepted
date: 2026-05-19
category: api-auth
---

# ADR-004: API Style and Auth — None

## Context and Problem Statement

The POC has no backend (per ADR-002, it is a static site) and no concept of a user account or session — Nora is the only intended user and she interacts with the installed PWA directly. The decision is whether the POC needs any API client surface or any authentication mechanism.

## Decision Drivers

- **POC scope is install + UI feel.** Auth and APIs are explicitly out of scope per the project brief.
- **No backend exists.** ADR-002 establishes a static site; there is nothing to authenticate against.
- **Privacy / compliance simplicity.** With no API and no auth, there is no PII handled, no session lifecycle to think about, no consent flows.

## Considered Options

1. **None — no API, no auth.**
2. **Read-only public REST endpoint** — for example, a mock MLS feed served from JSONPlaceholder or a static GitHub Pages JSON file.
3. **Anonymous session cookie (no auth, just identity)** — to demonstrate the `nestor_sid` cookie pattern from the full product.

## Decision Outcome

**Chosen option:** "None — no API, no auth", because the POC has no remote dependencies and no need for user identification. The full product's anonymous-session model (`nestor_sid`, see `../specs/001-search-browse/spec.md`) is documented for the future build but not exercised here.

### Consequences

- Good, because zero attack surface — no endpoints, no tokens, no cookies.
- Good, because no CORS, no auth library, no session management code.
- Good, because the POC operates fully offline after first load with no API to mock.
- Bad, because the POC does not exercise the eventual session model.
- Neutral, because no auth means no path for Nora to "log in" — but she doesn't need to.

## Pros and Cons of the Options

### None — no API, no auth (chosen)

- Good, because nothing to break and nothing to secure.
- Bad, because no validated learning about the auth flow.

### Read-only public REST endpoint

- Good, because would exercise the service worker's runtime caching behavior.
- Bad, because adds failure modes and zero validated learning about install/UX.

### Anonymous session cookie

- Good, because would prototype the eventual `nestor_sid` pattern.
- Bad, because there is no backend to issue or read the cookie — the value would be set client-side and serve no purpose.

## Constraints on Future Work

- The POC MUST NOT make any outbound HTTP calls except to CDN-hosted static assets (fonts, icons).
- The POC MUST NOT set, read, or rely on any authentication cookie, JWT, session token, or OAuth flow.
- Adding any API client, auth, or session concept requires a new change proposal that promotes the project beyond the POC stage.

## Deferred Aspects

None for the POC. The full product's REST + SSE contracts and Clerk-based auth handoff are documented in `../specs/001-search-browse/contracts/` and are explicitly out of scope here.
