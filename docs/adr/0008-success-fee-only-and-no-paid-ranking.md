# ADR-0008: a fee only on the first consultation of a pair, and no paid placement

- Status: accepted
- Date: 2026-11-12

## Context
The business analysis names leakage as the largest weakness: after the first conversation a client can contract the
practitioner directly. It also names payment for an outcome, not for advertising, as a real advantage.

## Decision
- The platform fee applies to the first completed consultation between one client and one practitioner, and never again for
  that pair. The rate is a parameter (D-23).
- Ranking cannot be bought. No input to the ranking formula is a payment, a plan or a promotion (SPEC §6.1, CONTRACTS §9.3).
- The ranking rules are public, versioned, and each version is published with its change date.

## Consequences
- The revenue per pair is bounded by the first consultation. Retention beyond it is a later product problem (Pro
  subscription, contracts, escrow), listed as a non-goal of the MVP.
- Fairness of ranking can be claimed truthfully, which is part of the trust the product sells.

## Alternatives considered
- A fee on every consultation: raises leakage pressure at once.
- Paid boosts: contradicts the claim that ranking follows the published rules.
