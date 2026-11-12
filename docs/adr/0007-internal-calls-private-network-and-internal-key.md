# ADR-0007: service-to-service calls use the private network and an internal key

- Status: accepted
- Date: 2026-11-12

## Context
authservice has no client-credentials grant, so a service cannot obtain a token of its own. Some calls are not made on behalf
of a user: a payment webhook reaching Booking, Reputation pushing a rating summary to Directory, Reputation asking Booking
whether a booking is reviewable.

## Decision
- Services other than the web app have no public listener in production. They are reached over the private network
  (Fly 6PN) only.
- Endpoints under `/internal/` additionally require the header `X-Internal-Key`, compared in constant time against a platform
  secret that the callee holds and the callers are given (one key per callee, not one for the estate).
- A call made on behalf of a user carries that user's token, forwarded by the web app, and is authorised by the callee.
- Payment webhooks are the exception: they come from the internet, are authenticated by the provider signature over the raw
  body (never by the internal key), and are exposed on a dedicated public path of Booking only (SPEC §6.3).

## Consequences
- A shared static key is weaker than a signed token. The risk is accepted for the MVP because the network is private and the
  key is per callee; rotation is a secret change plus a restart. A move to signed service tokens is a later decision.
- Local and fake profiles run the same check with a development key from the AppHost parameters.

## Alternatives considered
- One shared signing secret for all services: rejected by P5 (verify equals mint).
- Mutual TLS: rejected as disproportionate at this size (reference architecture §4: no service mesh).
