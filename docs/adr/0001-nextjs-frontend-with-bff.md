# ADR-0001: Next.js with a backend-for-frontend is the product surface

- Status: accepted
- Date: 2026-11-12

## Context
The product lives on public, indexable pages: category pages, practitioner pages, a ranking-rules page. Organic search is the
main acquisition channel (docs/research/BUSINESS-ANALYSIS.md §4). The standards offer two frontends: Next.js
(`FRONTEND-BFF`, the estate default for public, horizontally scaled products) and a Blazor Web App (`BLAZOR-WEB-APP`, for
bounded audiences, kept running because pages live in server memory).

## Decision
One Next.js app (`web/`, App Router, TypeScript, pnpm) is the only product surface. It follows `FRONTEND-BFF`: runtime
configuration through `/api/config`, HttpOnly cookie sessions, one catch-all proxy route, and edge middleware that verifies
the JWT signature against the JWKS of the identity service. Public pages are server-rendered and cacheable. The public site
carries no third-party scripts in the MVP.

## Consequences
- Public pages can be indexed, cached and scaled to several machines; a Blazor circuit per visitor would not suit crawlers
  and anonymous traffic.
- The repository gets a Node toolchain and a second test stack (vitest, Playwright). CI has a `web` job for it.
- TypeScript types for the services are generated from their OpenAPI documents, so contracts cannot drift silently
  (SPEC §5 rule 9).

## Alternatives considered
- Blazor Web App: simpler for a .NET-only team, rejected because the product is an anonymous, SEO-driven, scaled surface.
- Blazor WebAssembly: not supported by the standards.
