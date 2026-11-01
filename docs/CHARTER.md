# Moneytrail Charter

## Purpose

I am building this simulated payment platform to learn and demonstrate payment correctness, backend engineering, and
distributed-systems recovery.

## Core outcome

A reviewer must be able to see how a merchant payment moves from creation to capture, settlement, and payout. They will
also be able to see failures, their clear explanations and recovery.

## First release

The first release will include multiple synthetic merchants, one payment rail, auth, capture, ledger, settlement
reconciliation (recon), payouts, webhooks, and failure scenarios.

## Non-goals

We won't deliberately build real money, customer checkouts, cards, KYC/AML, fraud, FX, disputes, prod non-compliance
claims, and a Stripe clone.

## Success evidence

We will know this project worked successfully if:

- A clean checkout starts locally.
- A payment completes end-to-end.
- A lost response, duplicate event, and missing settlement line are reproducible.
- Ledger entries balance and merchant balance can be rebuilt.
- A new engineer can run the demo and explain the limits.

## Constraints

The limits of the system are:

- Includes only synthetic data and money.
- 