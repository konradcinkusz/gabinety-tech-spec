# ADR-0004: the LLM runs in its own service; the Copilot subscription never serves public traffic

- Status: accepted
- Date: 2026-11-12

## Context
Some features are better with a language model: turning a client's free-text problem into a structured brief, and triaging
reviews for a human moderator. The sibling project `llm-multi-agent-chess` shows how the GitHub Copilot SDK is used properly:
one project (`Chess.Llm`) is the only one that references the SDK, SDK types never leave it, sessions have no tools, every
call has a timeout and an abort, usage is tallied, a missing login degrades and never fails the host, and a viewer-paid
request never runs on the owner's subscription (its invariant `RequireApiKeyModels`).

A language model brings a credential, a cost and an availability risk (P10 corollary: that makes it a service, P3).

## Decision
1. `Gabinety.Llm` is the only project that references `GitHub.Copilot.SDK`. Its design is `docs/spec/COPILOT-SDK.md`.
2. The Assistant service (`Gabinety.Assistant`) is the only service that references `Gabinety.Llm`. Every other service calls it
   over HTTP and works without it (P8).
3. Traffic caused by the public (clients, practitioners, moderators of a deployed system) runs only on bring-your-own-key
   models (`SessionConfig.Provider`: Anthropic, OpenAI-compatible or Azure). The owner's Copilot subscription serves
   development, the live smoke run and evaluations run by the owner. This is enforced in code by
   `Assistant:RequireApiKeyModelsForTraffic` (default on), not by convention.
4. A model never decides anything about money, ranking, ratings, verification or the removal of content. It produces advice a
   deterministic rule or a person accepts (SPEC §5 rule 6).

## Consequences
- The Copilot terms of service for serving end users from a personal subscription are not verified here and are treated as
  forbidden (assumption A-04). The owner confirms or corrects this before any use beyond development.
- Whether a bring-your-own-key session also needs a Copilot runtime and sign-in is unverified (the chess project's open
  point). Spike T-01.02 settles it; until then the production image is assumed to carry the runtime (A-05).
- The AI features are optional: with the service or its provider down the web app shows the plain form.

## Alternatives considered
- Call a vendor SDK directly from Booking or Reputation: rejected (credential and availability risk in a core service).
- Use the Copilot subscription for all traffic: rejected (terms, rate limits, and the owner's personal quota).
