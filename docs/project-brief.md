# Project Brief: Nestor PWA POC

## Problem Statement

A small POC to give the client (Nora) the ability to test PWA technology on her end and experience it firsthand. The client needs to feel how the progressive web app behaves on mobile before committing to it as the delivery vehicle for the full product.

## Target Users

- **Primary:** Nora (client) — testing the PWA experience on her own device to validate the technology choice.
- **Secondary:** MoC internal team — demoing and QA-ing the POC before and during client review.

## Value Proposition

PWA can be added to the home screen with one tap — no App Store required. It behaves like a native app (offline splash, push-ready), while sharing the same codebase as the full web product. No separate native build needed, and Nora gets a real installable experience on her iPhone to evaluate before the full build.

## Core Features (v1)

1. Installable PWA shell — web app manifest + service worker enabling home-screen install on iPhone/Android with branded splash screen and offline fallback.
2. Listing cards + detail view — browsable property cards with photos and key facts, tappable to a full detail page with photo gallery.

## Scope Boundaries

### In scope

- PWA install flow: web app manifest, service worker, branded splash screen (iPhone Safari + Android Chrome)
- Static/mocked listing data: hardcoded fixtures, no real MLS integration required
- Mobile-first responsive UI shaped to the full product's design direction

### Out of scope

- AI chat agent (no LangGraph, no LLM calls)
- User auth / accounts (no Clerk, no registration, no login)
- Favorites persistence (no backend, no session storage)
- Production infrastructure (no ECS, RDS, ElastiCache — local or throw-away URL only)

## Business Constraints

- **Timeline:** ASAP — no hard date; ready when presentable to Nora.
- **Team:** Vlad Yermolin (Solution Lead), Tanya Chebanyuk (AI/prompt), Sasha Ocheretny (fullstack), Ivan Pavlovskyi (QA).
- **Budget:** No separate budget; part of the existing engagement.
- **Compliance:** None — this is a pure technology demo.

## Open Questions

- Which device/browser to prioritize? PWA install behavior differs between iPhone Safari and Android Chrome. Nora's primary device is unknown — need to confirm before finalizing manifest and service worker configuration.
