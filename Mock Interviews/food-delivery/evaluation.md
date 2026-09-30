---
title: "Design a Food Delivery Platform — Evaluation and Scoring"
status: active
date: 2026-09-29
tags: [hld, mock, food-delivery, evaluation]
---

# Design a Food Delivery Platform — Evaluation and Scoring

Session reviewed against the five phases in [[problem|Problem Statement]] and [[06-hld-interview-checklist|HLD Interview Checklist]].

## Scoring

| Phase | Score | Why |
|-------|-------|-----|
| 1. Requirements Clarification | 10/10 | Opened on when the card is charged and immediately derived the failure story from the answer: authorize at checkout, capture at acceptance, void on reject, so an unresponsive restaurant is a free instant void rather than a fee-bearing refund. Then asked about partner offer exclusivity, which reframed dispatch as a compare-and-set, and about mid-checkout price changes, which produced the order-line snapshot. |
| 2. Scale Estimation | 9/10 | Peak orders at 667/s, catalog at 18K reads/s or 500K items/s, partner pings at 400K/s, tracking fan-out at 1.2M msg/s, and 2.1K third-party payment calls/s. The 700x ratio between catalog reads and order writes is the number that justifies the caching strategy, and calling out the payment-processor rate limit as the external dependency is a strong read. Lost a point for not sanity-checking the 20 menu-views-per-DAU assumption against a second method. |
| 3. API and Data Design | 10/10 | Three BFFs justified on read volume and release cadence rather than on microservices fashion. The `UNIQUE (customer_id, idempotency_key)` on orders, `expected_from` compare-and-set transitions, and composed notification idempotency key are all the right mechanisms in the right places. Inventory as a reservation ledger with a materialized availability projection is a genuinely strong departure from the obvious decrementing counter. |
| 4. High-Level Architecture | 9/10 | Four planes drawn with different scaling characteristics, transactional at 700 writes/s, catalog at 18K cacheable reads, location at 400K lossy pings, realtime at 1.2M messages/s. The 12-step order flow with a millisecond timeline, and the four-planes framing, made the boundaries legible. Points lost for treating the ETA service as a consumer without showing where its model output is cached. |
| 5. Trade-offs and Failure Scenarios | 10/10 | Authorize-versus-capture, reservation-versus-counter, and greedy-versus-optimal dispatch were all argued with the cost named on both sides. All six pushbacks answered: 5x flash spike (pre-scale on leading indicator, queue and shed reads, async authorize, never shed the inventory check), primary death (ledger decoupled, client-side key retry, 90s RTO), bestseller hot key (insert-only holds, pre-projection), replica lag (no read at all, partial unique index, 60s reconciliation), and menu cache stampede (warm-then-flip as a two-phase publish). Ending with the compound failure, where push failure pushes 100K customers onto the order-status poll path, was the standout moment. |
| **Overall** | **49/50** | Strong hire. The payment-timing reasoning and the inventory reservation model are the two answers a hiring manager would actually remember. |

## What Made This a Strong Answer

- Asked when the card is charged before anything else, then treated the answer as an architectural input rather than a detail. The insight that an unresponsive restaurant becomes a void instead of a refund is the kind of thing that takes a year of production incidents to learn.
- Modeled inventory as reservations plus a materialized projection, which removes the platform's most contended single row at the exact moment demand peaks, and then named the cost honestly: a one-second-stale availability read that can oversell by a handful of items, compensated by cancellation and refund.
- Refused the dual write in both of its bad forms. Synchronous calls inside the transaction make a notification outage an order outage, and commit-then-call loses side effects on deploy. The outbox converts an unsolvable problem into an at-least-once one.
- Separated the dispatch hot spots instead of merging them. The Kafka partition keyed on restaurant id, and the CAS contention on 12 partner rows, are two different problems with two different fixes, and the pre-positioning mitigation is the only one that adds supply rather than redistributing it.
- Answered replica lag by removing the read. Verification and claim being the same statement on the primary means there is no window for lag to exploit, and the partial unique index means a bug in application logic still cannot produce a double assignment.
- Named the compound failure that nobody designs for: push silently drops, 100,000 customers fall back to polling, and the order-status read path becomes the outage. The answer is a separate cached read model with a rate limit, not the transactional path.

## Memory Hooks

- Authorize at checkout, capture on restaurant accept, void on reject. That one decision turns the most common operational failure into a free instant void instead of a slow fee-bearing refund.
- Three BFFs because read volume and release cadence differ, not because microservices. Menu browsing at 18K reads/s and a restaurant's order queue are different systems wearing one backend.
- 700 order writes/s, 18K catalog reads/s, 400K location pings/s, 1.2M tracking messages/s. The database is the easy part; catalog reads and the realtime plane are the system.
- Catalog read-to-write ratio is 240,000 to 1. That number is the whole reason the menu cache can be aggressive and the menu store can be its own thing.
- Available = capacity minus active holds, with a materialized count for reads. Insert-only beats a decrementing counter when the hot item is a single row at peak demand.
- The outbox: order write and event write in one local transaction, everything downstream an idempotent at-least-once consumer. Per-order ordering via a sequence column, never via a global key.
- Idempotency has four layers: unique constraint on the order insert, order id as the processor key, dedupe table for processor webhooks, and unique txn on ledger entries. Plus a poll-based reconciliation because "never delivered" and "very late" look identical.
- Greedy batch dispatch is 70 to 80 percent of optimal for microseconds; the assignment solve buys the last 20 to 30 percent with a whole tick of decision latency, which in food delivery is a product cost.
- Cache TTL must never exceed the validity of the thing cached. Fee quotes carry the surge version they were priced with, so a surge change invalidates by construction rather than by a sweep.
- ETA is max of prep and travel, not the sum, and the courier cannot move before the food is ready. That one change usually cuts the displayed ETA by 20 minutes.

## Weak-Slot Pointers

- Sharding was asserted, not designed. 256 order shards and geocell routing for partners were named, but you never justified the count, never said which key the orders table shards on versus how a restaurant's queue is read, and never addressed rebalancing. This is the single most common follow-up in a system-design follow-up round. Drill [[shard-key|Shard Key]] and [[shard-rebalancing|Shard Rebalancing]].
- The ETA service was described in one paragraph at the end, and it is a more interesting problem than it looks: a nightly retrained prep-time model per restaurant per item category, a historical median as a sanity floor, a maps provider with live traffic, and a fallback when the model is cold. Expect "how do you predict prep time" as the follow-up. Drill [[search-ranking|Search Ranking]] and [[time-series-at-scale|Time Series at Scale]].
- Menu modifiers were acknowledged as a schema complication and then dropped. Modifier groups with min/max select, price deltas, and their own inventory, plus time-limited items, are where menu schemas actually break, and it is a legitimate Phase 4 deep dive you skipped. Drill [[data-model-types|Data Model Types]] and [[normalization-vs-denormalization|Normalization vs Denormalization]].
- Multi-region was not addressed at all. You said the order is written in the restaurant's home region, which implies a correctness model, and then never stated what happens to a customer in another region, where the restaurant's data lives, or how data residency affects the ledger. Drill [[multi-region-models|Multi-Region Models]] and [[data-residency|Data Residency]].
- Compensation logic was strong for payments but thin for supply. There is no story for the partner who accepts and then their vehicle breaks down, for the restaurant that accepts 200 orders and can only cook 50, or for a delivery that goes to the wrong address. Each needs a ledger transaction, not a status change. Drill [[saga-and-strangler|Saga and Strangler]] and [[exactly-once-effect|Exactly Once Effect]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the five-phase skeleton this session was scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before the interview
- [[problem|Problem Statement]] — the prompt, clarifying questions, and evaluation criteria
- [[interview|Interview Study Transcript]] — the full dialogue
- [[outbox-pattern|Outbox Pattern]] — the dual-write fix
- [[idempotency|Idempotency]] — order, payment, webhook, and ledger correctness
