---
title: Payment System Design - Problem Statement
status: active
tags: [hld, mock, payment-system]
---

# Payment System Design

## Problem Statement

> Design the payment processing platform behind a large consumer application — think a super-app that sells digital goods, rides, food delivery, and subscriptions, settling in about forty countries.
>
> A customer can pay with a card (we do not store raw card numbers; we tokenize through a third-party payment service provider), a stored wallet balance, or a bank debit mandate. The application we are building sits between the customer and one or more external payment service providers, commonly referred to as a PSP.
>
> In scope:
> - Creating a payment intent and authorizing it against a funding source
> - Capturing an authorized amount, sometimes partially, sometimes later than the authorization
> - Holding funds (escrow) for orders that are not yet fulfilled, and releasing or forfeiting them
> - Full and partial refunds, refund to a different instrument, and refund failure handling
> - Wallet top-ups and wallet-to-wallet transfers
> - An internal accounting ledger that can answer "what is my balance" and "show me every movement of every dollar" without trusting application logs
> - Asynchronous fraud scoring that can hold, block, or step up a transaction after it has already been accepted
> - Outbound webhooks to merchants informing them of terminal state changes, with retries and delivery tracking
> - Daily reconciliation against the PSP statement, and a settlement report per merchant
> - A chargeback and dispute intake path
>
> Out of scope:
> - Actual card network rails, EMV, and 3-D Secure protocol internals
> - Building a banking core or holding customer deposits under a banking licence
> - Currency FX trading, hedging, and treasury operations beyond selecting a conversion rate snapshot
> - Storing, tokenizing, or vaulting raw PANs; that is explicitly the PSP's job
>
> Hard requirements you must respect:
> - **Money is never a float.** All amounts are integer minor units plus an explicit currency code.
> - **The ledger is the source of truth**, not the payment table, and not the balance cache.
> - **A customer retry must never double-charge.** Network timeouts create duplicate attempts and you will be asked how you prevent that.
> - **The system must be auditable.** Every state change has an actor, a timestamp, and a reason, and nothing in the audit log is ever updated or deleted.
> - Losing a single write is unacceptable. Losing read freshness for a few seconds is fine.

---

## Clarifying Questions You Should Ask

Strong candidates ask these before drawing anything. Pick the ones you genuinely need answered.

**On money semantics**
1. Is this marketplace-style — do we hold funds for a period before paying the merchant out, or is authorization and immediate capture the norm?
2. Do we keep balances, or just movements? Is the customer balance a stored column or always derived by summing the ledger?
3. Can one payment be captured in several chunks, and can a refund be larger than what has been settled?
4. Which currencies, and how do we pick a FX rate — at authorization time, at capture time, or at settlement? Does the rate need to be snapshotted and locked?
5. What is the exact meaning of a day for the purposes of the settlement report, and whose timezone?

**On correctness and failure**
6. What is the required guarantee on the ledger — is a single missing movement a legal or existential problem?
7. When the PSP call times out, is the transaction "maybe succeeded"? How long are we willing to leave a payment in an unknown state?
8. How often is a customer expected to tap retry? What does our own mobile client do on timeout, and can it generate two requests in five hundred milliseconds?
9. What is the reconciliation cadence, and what does the PSP actually give us — a per-transaction file, or only a daily aggregate?
10. Is the fraud decision allowed to be made after authorization, and if so, how do we reverse a payment that fraud has already killed?

**On scale and money flow**
11. DAU, payments per user per day, and the peak-to-average ratio during payday and festival windows?
12. What fraction of transactions are retries from client timeouts rather than fresh attempts? This number changes the write amplification by an order of magnitude.
13. Average and p99 ticket size, and what is the top merchant by volume? A single hot merchant will dictate your shard key.
14. How many merchants, and do they each need their own webhook endpoint with its own delivery SLO?
15. What is the acceptable cost ceiling per transaction? Webhook fanout, per-request PSP calls, and ledger writes all cost money.

**On compliance and operations**
16. What is our regulatory posture — do we need to avoid touching PAN data entirely, which changes the architecture at the top?
17. How long do we retain transaction and audit records, and does anything require immutable storage or WORM semantics?
18. Who consumes reconciliation mismatches, and how fast? Seconds or a daily batch?
19. What is the availability target, and is a payment a 99.9% path or a 99.99% path? The two justify very different spend.
20. Is multi-region active-active in scope, or is a single region with a tested recovery story acceptable for now?

---

## What You Are Evaluated On

### Phase 1: Requirements
Separate functional from non-functional. Nail the money semantics before anything else: who holds funds, when capture happens, what a refund is against, and what the ledger is authoritative for. State your assumptions rather than silently absorbing them.

### Phase 2: Estimate
Do the math out loud with real numbers: DAU to payments per day, average and peak TPS, write amplification from retries, in-flight concurrency against a slow PSP, bytes per payment, total ledger growth per year, and the resulting storage after retention. Show the arithmetic, not just the conclusion.

### Phase 3: High-level design
Draw the architecture. Identify the payment orchestration service, the ledger service, the PSP adapter layer, the fraud pipeline, the webhook dispatcher, the reconciliation job, and the read-model store. Name the shard key for each store and defend it.

### Phase 4: Deep dive
The expected core is the payment state machine, the idempotency contract on every money-moving endpoint, double-entry bookkeeping, and the dual-write problem solved by the outbox pattern. Be ready to talk through webhook retry semantics, ambiguous PSP timeouts, and why fraud is asynchronous.

### Phase 5: Trade-offs and follow-ups
Defend the choices. "Why not two-phase commit with the PSP", "why not a distributed lock per customer to serialize charges", "why is the ledger not just a column on the payment row", "what if the primary database dies mid-capture", "how do you survive a hot merchant and replica lag on the ledger". Know exactly which trade-off you accepted and what it cost.
