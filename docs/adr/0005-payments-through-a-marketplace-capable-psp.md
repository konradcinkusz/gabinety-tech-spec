# ADR-0005: payments go through a marketplace-capable payment provider; the platform never holds funds

- Status: accepted
- Date: 2026-11-12

## Context
A consultation is paid by the client and earned by the practitioner; the platform keeps a fee on the first consultation. Holding
other people's money can be a regulated activity. The estate's `PAYMENTS-AND-MONETIZATION` guide is written for a SaaS with a
merchant of record, not for a marketplace, but its webhook, idempotency and mock-first rules apply unchanged.

## Decision
- One internal abstraction `IPaymentProvider` (P11), one adapter per vendor, no vendor type crossing the boundary.
- The real adapter is a provider that supports split payments or connected accounts, so funds go to the practitioner and the
  fee to the platform without the platform holding balances. Which provider is an owner decision (D-26) and a later ADR.
- Mock-first: the whole path (checkout, webhook, refund, payout) works with `Payments:Provider=Mock` and zero credentials. The
  simulate endpoint answers 403 unless the provider is Mock.
- Money is an integer number of grosz plus a currency code. No floating point anywhere in money code.
- Whether this structure is sufficient in law, and how invoices for the fee are issued, are not decided here (A-06, A-07).

## Consequences
- M1 is reachable on the mock; M2 needs the owner's provider account (D-08).

## Alternatives considered
- Platform-held escrow account: rejected for the MVP (regulatory exposure).
- Pay the practitioner directly and invoice the fee later: rejected, it recreates the leakage problem on the first booking.
