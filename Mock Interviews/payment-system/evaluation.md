---
title: Payment System Design - Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, payment-system, evaluation]
---

# Payment System Design — Evaluation

Session: 45 minutes of design conversation. Interviewer: Principal Engineer, Payments.
Scale assumption used: 50M registered, 15M DAU, 30M payments/day, 15% client retry rate, 4x peak.

## Scoring

| Phase | Score | Why |
|---|---|---|
| Phase 1 — Requirements | 4.5 / 5 | Opened on money semantics rather than architecture, pinned hold-versus-capture, and stated the ledger-is-truth asymmetry before drawing a single box. |
| Phase 2 — Estimate | 4.5 / 5 | Arithmetic shown at every step (347 avg / 1,400 peak TPS, 2,800 in-flight PSP calls, 150 GB/day, 55 TB/year) and the in-flight concurrency number correctly forced the async contract. |
| Phase 3 — High-level design | 4 / 5 | Clean nine-component diagram with named shard keys per store, though the read model and archive lanes were added late rather than designed in from the start. |
| Phase 4 — Deep dive | 5 / 5 | Outstanding on the state machine including an explicit `unknown` state, three-layer idempotency, double-entry with correct debit/credit signs, and the outbox with a partial index for the relay. |
| Phase 5 — Trade-offs | 4.5 / 5 | Refused 2PC, per-customer locking, and synchronous fraud with structural rather than probabilistic reasoning, and every accepted cost was named explicitly. |
| **Overall** | **4.5 / 5** | Strong hire signal for Senior, comfortably above the bar for Staff on the correctness and audit axes. |

## What Made This a Strong Answer

- **The 2,800-concurrent-calls number.** Deriving that a 1,400 TPS peak against a 2-second PSP dwell needs 2,800 in-flight outbound connections is what justified the fully asynchronous 202-plus-status-URL contract. That single calculation changed the whole API shape, and most candidates never get there.
- **Naming the ambiguous outcome as a real state.** Putting `unknown` in the state machine with a resolver as the only actor permitted to exit it is the difference between a payment system that recovers and one that either double-charges or loses payments.
- **Sharding the journal, not the account.** The candidate walked through why sharding `ledger_entries` by `account_id` would split a journal's postings across shards, then chose `journal_id` as the shard unit and made balances an async projection. That is the single hardest idea in the problem and it was handled cleanly.
- **Idempotency explained in three layers, each covering a different crash window.** Key uniqueness in my store, a persisted PSP key generated *before* the outbound call, and unique journal ids. Stating that level one alone fails when the process dies between commit and PSP call is the kind of precision interviewers listen for.
- **"Effectively-once business effect" instead of "exactly-once delivery."** The candidate explicitly refused the overclaim and named the actual mechanism: at-least-once from the outbox relay plus idempotent consumers, with compare-and-set on the payment state as the exactly-once enforcement point.
- **Fail closed on money, fail open on reads.** Database failure stops accepting payments but reads degrade to the read model rather than erroring. Stated as a deliberate, money-domain-specific choice.

## Memory Hooks

- Authorize is not a state, it is a liability: one journal, postings sum to zero, always.
- Unknown is a state. Resolver is the only actor that leaves it.
- Outbox row in the same transaction as the business row — the DB and the broker can never disagree.
- At-least-once plus idempotent consumers equals effectively-once effect. Say "effectively-once", never "exactly-once delivery".
- 1,400 peak TPS times 2-second PSP dwell equals 2,800 in-flight calls, which means 202, not 200.
- Shard the journal so postings stay atomic; make balances a projection so reads stay O(1).
- Debit reduces a liability, credit reduces an asset. A refund is a new journal, never an edit.

## Weak-Spot Pointers

- **Settlement and payouts were designed around but not designed in.** Merchant payout scheduling, reserve/holdback, negative balances on merchants, and the multi-currency payout batch were hand-waved. Drill: [[transactions-and-acid|Transactions and ACID]] and see whether you can define a payout run end to end.
- **Reconciliation was named but not specified.** "Three-way match on reference, amount, fee" is a slogan. You should be able to state the match keys, the break reason codes, the storage model, the partial-match case, and the SLI. Drill: [[event-sourcing-cqrs|Event Sourcing and CQRS]] and [[outbox-pattern|Outbox Pattern]].
- **Multi-region was dismissed too quickly.** Quorum writes in-region, async replica across regions, and "RPO zero" asserted without reconciling that with the asynchronous cross-region copy. Have a crisp answer for whether the cross-region path is active-active reads, warm standby, or nothing. Drill: [[database-replication|Database Replication]] and [[rpo-rto|RPO and RTO]].
- **The PSP outage story is thinner than the database outage story.** No numbers on circuit-breaker thresholds, connection-pool sizing against the PSP, or how the command-worker backlog drains. Drill: [[circuit-breaker|Circuit Breaker]] and [[retry-and-timeout|Retry and Timeout]].
- **Cost estimation appeared only for storage.** No cost per transaction, no revenue-per-transaction sensitivity, and no argument for what the 15% retry rate costs. Drill: [[cost-estimation|Cost Estimation]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — walk the seven phases and check the estimate section was actually covered.
- [[01-rapid-revision|Rapid Revision]] — re-read the money-semantics and consistency rows before the next mock.
- [[outbox-pattern|Outbox Pattern]] — the deepest dive in this session, worth re-reading properly.
- [[exactly-once-effect|Exactly-Once Semantics]] — the terminology distinction this session hinges on.
- [[distributed-transactions|Distributed Transactions]] and [[saga-and-strangler|Saga Pattern]] — the alternative the candidate rejected, so you can defend the rejection.
