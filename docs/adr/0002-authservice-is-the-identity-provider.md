# ADR-0002: authservice is the only identity provider; marketplace roles live in Directory

- Status: accepted
- Date: 2026-11-12

## Context
The estate runs one open-source identity service, `konradcinkusz/authservice`: accounts, RS256 JWTs with a JWKS endpoint and
OIDC discovery, refresh rotation, GitHub and Google sign-in, consent records, data export and erasure, an admin API. It is
not a general OIDC provider: no client-credentials grant, no ID tokens, no userinfo endpoint. Its organization roles
(`Owner`, `Admin`, `Member`) and its platform roles (`Admin`, `SuperAdmin`) do not describe a marketplace.

## Decision
- Product services keep no user store, no password and no token minting. They validate tokens against authservice's JWKS.
- authservice runs from its published container image as its own app. Its source is never modified here.
- The meaning of "practitioner" and "moderator" is owned by the Directory service, keyed by the token subject (`sub`).
  "Administrator" is the authservice platform role `Admin` or `SuperAdmin`, read from the token.
- GitHub sign-in through authservice is how a practitioner proves control of a GitHub account (SPEC §6.1 verification).
- Service-to-service calls do not use authservice tokens (ADR-0007).

## Consequences
- A role change for a practitioner or moderator is effective at once (a row in Directory), not at the next token refresh.
- The exact claim names and endpoints are an assumption until spike T-01.01 reads them from the authservice source (A-01).
- The consent and erasure endpoints of authservice are used as they are. Directory, Booking and Reputation each expose an
  internal erasure hook for the subject (SPEC §11).

## Alternatives considered
- A user table in Directory: rejected, it is a second identity system (P5).
- Marketplace roles as authservice organization roles: rejected, the model does not fit and would couple the product to the
  identity service's schema.
