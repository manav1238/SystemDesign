---
title: Stock Trading Platform - Interview Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, stock-trading-platform, evaluation]
---

# Stock Trading Platform - Interview Evaluation

**Problem:** Retail brokerage backend with live quote streaming and order execution
**Duration:** 45 minutes
**Overall:** Very strong. The candidate correctly identified that the matching engine is not the bottleneck, which is the single biggest differentiator in this problem.

---

## Scorecard

| Phase | Score | Why |
|---|---|---|
| Phase 1: Requirements | 9/10 | Separated exchange from broker, pinned down what "no double execution" meant, and pushed back on the 99.99 percent requirement by reframing it as per-symbol availability. |
| Phase 2: Estimation | 9/10 | Showed the uncapped fanout number (120M msg/s) and then the coalesced number (24M msg/s) with the 5x reduction justified by human perception, not just by convenience. |
| Phase 3: High-level design | 8/10 | Two planes with a deliberate absence of dependency between them; slightly rushed on the account and position service, which deserved its own diagram rather than a box. |
| Phase 4: Deep dive | 9/10 | Idempotency as a single unique constraint, fill caps enforced inside the single-threaded critical section, and snapshot-plus-delta recovery with determinism discipline were all exactly right. |
| Phase 5: Trade-offs | 9/10 | Named six real concessions including symbol migration difficulty and at-least-once delivery, and argued fail-closed on risk versus fail-open on availability with a reason. |

**Total: 44/50**

---

## What Made This a Strong Answer

- **Named the non-bottleneck first.** 30,000 orders per second against a single-threaded engine that handles a million is a rounding error, and saying "the matching engine is not the bottleneck, do not spend the interview here" redirects all the effort to intake, fanout, and durability, which is where it belongs.
- **Treated per-symbol single-writer as the algorithm, not a limitation.** One symbol, one thread, one total order, no locks, no consensus on the hot path. That framing turns a sharding question into a correctness argument, which is the correct framing.
- **The idempotency mechanism was one database constraint.** A unique-constrained insert that returns the stored prior response is the entire no-duplicate-order story, and resisting the urge to build something more elaborate is a sign of judgment.
- **Over-fill was made structurally impossible.** Fill quantity bounded by remaining quantity computed inside the same critical section as the mutation means the guarantee is a property of the design rather than a check that could be bypassed. This is the difference between a rule and an invariant.
- **Availability was decomposed into per-symbol units.** Three separate availability domains, matched, intake, and quotes, with a deliberate statement of what survives each failure. That is a better answer than any percentage.
- **The quote and order planes were made independent.** Displaying exchange quotes rather than deriving them from the internal book means a matching outage never becomes a quoting outage. The candidate stated this as a deliberate availability choice rather than a detail.

---

## Memory Hooks

- Single-threaded per symbol. One symbol, one thread, one total order. Scale by adding symbols.
- Over-fill is impossible because the cap lives in the same critical section as the mutation.
- Idempotency is a unique constraint plus replay of the stored response. That is all it is.
- Coalesce quotes to 20 per second per connection. It is a perception limit, and it is 5x cheaper.
- The log is the source of truth; the book and positions are rebuildable views.
- Fail closed on risk, fail open on availability. Cancels outrank new orders under load.
- acks=all with min.insync.replicas=2, and epoch fencing so a zombie leader cannot come back.

---

## Weak-Spot Pointers

- **The idempotency table's write amplification was mentioned but not sized.** 36M orders per day, retained 24 hours, 400 bytes a row, is roughly 350 GB of writes per day on a table that is pure overhead. The candidate should have proposed a hot store with a TTL plus a durable audit record, rather than treating one table as both. Drill [[outbox-pattern|Outbox Pattern]] and [[idempotency|Idempotency]] for the correct split between a dedupe key store and an audit log.
- **At-least-once versus exactly-once was asserted without the failure walk-through.** Dedupe on symbol plus sequence sounds right, but the hard case is a consumer that crashes after applying and before committing its offset, and the harder case is a rebalance where two consumers briefly own the same partition. Drill [[exactly-once-effect|Exactly-Once Effect]] and be able to narrate that interleaving.
- **Leader election and fencing were hand-waved.** "Epoch fencing" was named but the mechanism was not described, and a symbol shard failover is the highest-stakes failover in the system. Drill [[split-brain|Split Brain]] and [[raft-and-paxos|Raft and Paxos]] for term-based leadership and how a recovering replica is prevented from accepting stale writes.
- **Quote coalescing had no stated staleness bound.** Delivering at most 20 updates per second is correct, but the answer never said what the worst-case display staleness is when a symbol is ticking 200 times a second and the user is looking at order book state for trading decisions. That needs a number. Drill [[tail-latency|Tail Latency]] and think about the interaction between coalescing interval and decision quality.
- **No reconciliation story between the position shadow and the account ledger.** The answer correctly replays forward from the log, but never defined who runs the reconciler, at what cadence, and what a mismatch means operationally. Drill [[replayability|Replayability]] and [[event-sourcing-cqrs|Event Sourcing and CQRS]] for the projection rebuild and drift-detection pattern.

---

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] - the phase structure this transcript follows
- [[01-rapid-revision|Rapid Revision]] - the one-page recall sheet for ordering, durability, and fanout decisions
- [[kafka-ordering|Kafka Ordering]] - per-key ordering and the fanout implications
- [[publish-subscribe|Publish Subscribe]] - the quote fanout plane in isolation
- [[websockets|WebSockets]] - a million connections, backpressure, and reconnect
- [[hotspot-handling|Hotspot Handling]] - one symbol taking 20 percent of all orders
