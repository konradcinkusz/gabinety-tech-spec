# ADR-0003: a development-only identity stand-in for the fake profile

- Status: accepted
- Date: 2026-11-12

## Context
The fake profile (SPEC §4.1) must start the whole system without Docker, without accounts and without credentials, and the
end-to-end journeys must run in it. authservice needs PostgreSQL or SQL Server, so it cannot run in that profile.

## Decision
`tools/Gabinety.DevIdentity` is a small ASP.NET Core app that exposes the subset of the authservice HTTP surface the web app
uses (register, login, refresh, logout, me, JWKS, OIDC discovery) and issues RS256 tokens from an in-memory user list seeded
with synthetic users. Rules:
- it lives under `tools/`, is referenced by no project under `src/`, and is in no Dockerfile (architecture test);
- it refuses to start unless `GABINETY_FAKE=1`;
- the real surface is proven separately: CI starts the real authservice image and runs the same auth contract tests against it
  (T-00.04, T-03.01), so the stand-in cannot drift unnoticed.

## Consequences
- This is a deliberate exception to "services do not mint their own tokens": the tool is not a service of the product and
  never runs outside the fake profile and the end-to-end job.
- A result obtained against the stand-in is reported as such (SPEC §7).

## Alternatives considered
- Run authservice with Docker everywhere: rejected, the fake profile and cloud implementer sessions would need Docker.
- Skip authentication in the fake profile: rejected, role-based behaviour (practitioner, client, admin) is the product.
