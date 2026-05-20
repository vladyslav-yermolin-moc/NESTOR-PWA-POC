---
status: accepted
date: 2026-05-19
category: deployment
---

# ADR-005: Deployment Model — GitHub Pages

## Context and Problem Statement

Nora needs an HTTPS URL she can open on her iPhone and add to her home screen. Service worker registration is blocked on plain HTTP, so HTTPS is non-negotiable. The site is a folder of static files (per ADR-002) with no build step (per ADR-001), so any static host that serves HTTPS works. The decision is which host to use.

## Decision Drivers

- **HTTPS required.** Service workers will not register over plain HTTP except on `localhost`.
- **Zero infra ownership.** The POC is throw-away; we want no AWS account, no DNS work, no certificate management.
- **Shareable URL.** Nora needs a single link that works without any installation steps on her side beyond "Add to Home Screen".
- **No build step.** GitHub Pages serves a repo's static files directly — matches ADR-001.

## Considered Options

1. **GitHub Pages** — push static files to a repo, enable Pages, share the `*.github.io` URL.
2. **Cloudflare Pages / Netlify / Vercel static** — git-connected or drag-and-drop static hosting.
3. **Local dev only (`localhost:8000`)** — share screen instead of a URL.
4. **AWS S3 + CloudFront** — what the full product will use per the parent CLAUDE.md.

## Decision Outcome

**Chosen option:** "GitHub Pages", because it requires zero infra ownership beyond a GitHub repo (which we already have for the code anyway), serves HTTPS by default on `*.github.io`, and has no build step requirement. The POC is throw-away and does not justify standing up AWS infrastructure.

### Consequences

- Good, because the deployment process is `git push` to a configured branch — no CI required.
- Good, because HTTPS is automatic on the `*.github.io` URL — no certificate work.
- Good, because the URL is shareable as-is with Nora.
- Bad, because the URL contains the org and repo name, which is not brandable — acceptable for a POC.
- Bad, because GitHub Pages serves under a subpath (`/<repo>/`), which means all `manifest.webmanifest` paths and the service-worker `scope` must be relative or include the subpath. This is a real constraint and is enforced below.
- Neutral, because the POC will not use a CI build pipeline — files are committed pre-built.

## Pros and Cons of the Options

### GitHub Pages (chosen)

- Good, because zero infra, HTTPS by default, free.
- Bad, because subpath serving requires relative paths everywhere.

### Cloudflare Pages / Netlify / Vercel

- Good, because root-domain serving (no subpath), instant deploys on push.
- Bad, because requires another account and adds an extra integration to set up.

### Local dev only

- Good, because nothing to deploy.
- Bad, because Nora cannot install the PWA on her phone from a localhost URL — defeats the entire purpose.

### AWS S3 + CloudFront

- Good, because matches the full product's eventual deployment.
- Bad, because requires AWS account, IAM, bucket policy, CloudFront distribution, ACM certificate — drastically overengineered for a POC.

## Constraints on Future Work

- The build artifact MUST be a folder of static files committed to the repo. No GitHub Actions build step is required for the POC.
- The URL given to Nora MUST be HTTPS (GitHub Pages provides this by default). Plain HTTP MUST NOT be used.
- All paths in `manifest.webmanifest`, the service worker's `scope`, `start_url`, and any `<link rel="manifest" href="...">` reference MUST be relative or include the GitHub Pages repo subpath. Absolute paths starting with `/` MUST NOT be used unless they include the subpath.
- The service worker file MUST be served from the same scope as the app shell — placing it in a `js/` subdirectory would narrow its scope and break installability.
- Custom domains are out of scope for the POC. The default `<org>.github.io/<repo>/` URL is acceptable.

## Deferred Aspects

Production deployment (AWS S3 + CloudFront for the full product) is documented in the parent project's `CLAUDE.md` and is explicitly out of scope here.
