---
title: Kafka Replication
category: Messaging
priority: must-know
status: learning
difficulty: medium
interview_ready: false
tags:
  - hld
  - messaging
  - kafka
---

# Kafka Replication, Leader/Follower, and ISR

## 1. One-Line Definition
Kafka replicates every partition to a leader and follower replicas on different brokers; the leader serves reads/writes, followers copy the log, and only followers in the ISR (in-sync replicas) can take over leadership — which is exactly what governs how much data loss and unavailability a cluster permits.

## 2. Why Do We Need It?
A partition on one broker is a single point of failure and a storage ceiling. Replication keeps N copies so a broker crash doesn't lose data or availability. But naive copying is dangerous: a stale replica taking over leadership could *reorder or lose* the tail. The ISR model fixes a leader election to only accept replicas that are actually in sync — durability is a *consequence* of who is allowed to be leader, not an afterthought.

## 3. Simple Intuition
A team's work ledger with copies at three offices (brokers). The **leader** holds the master copy and is the only one who accepts edits; the two **followers** copy every new page. The team can only promote a follower who has *all* pages up to the last edit tick — the "in-sync" read of the day (ISO date). If the leader's office burns down, an in-sync follower takes over and no page is lost. If a follower is sick for days (behind), it can't be promoted — better to keep serving from a healthy copy (availability) than promote a stale one that would erase recent edits (loss).

## 4. What Happens Without It?
Single-copy partitions: broker crash = permanent data loss. Naive multi-copy with last-writer-wins: transient network glitch makes a stale replica "the latest," silent data loss and corrupted ordering. Choosing between those is why prevailing design is: **replication + acks + ISR-based failover** with an explicit "prefer availability or prefer durability" switch.

## 5. Core Idea
- **Per-partition replication:** replication factor N → N brokers hold the partition. One **leader**; N−1 **followers**.
- **Followers are log fetchers:** they pull from the leader and replay to their own log — they do *not* serve reads or writes.
- **ISR (in-sync replicas):** the set of followers caught up with the leader's high watermark within `replica.lag.time.max.ms` (and, earlier, configs like `min.insync.replicas`). Only ISR replicas may:
  - become the new leader, and
  - participate in the acks=all write acknowledgment.
- **High watermark:** last offset all ISR replicas hold — the only tail consumers may read. `LEO` (log end offset) = leader's newest append; gap = unwritten tail.
- **Leader election:** controller watches; on leader failure it elects from ISR. If ISR is empty:
  - `unclean.leader.election.enable=false` (default): no leader → *unavailable* but no data loss.
  - `true`: elect an out-of-sync replica → stays available but may drop the tail (loss).
- **acks=all + min.insync.replicas:** writes only ack when `minISR` copies ack → if ISR < minISR, producers get NOT_ENOUGH_REPLICAS (writes rejected to avoid pretending durability).

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Leader | Partition replica that serves reads/writes |
| Follower | Replica that copies from the leader |
| ISR | In-sync replicas, eligible to lead/ack |
| High watermark | Committed tail replicated to all ISR |
| LEO | Newest offset appended to the leader's log |
| Replication factor | Number of copies of a partition |
| min.insync.replicas | Minimum ISR size to accept writes |
| Unclean election | Promoting an out-of-sync replica (loss) |
| Controller | Node that orchestrates leader elections |

## 7. Basic Architecture

```mermaid
flowchart LR
    P[Producer acks=all] -->|append| L[Partition leader - Broker 1]
    L -->|E[fetch/replicate]| F1[Follower - Broker 2 - ISR]
    L -->|replicate| F2[Follower - Broker 3 - ISR]
    L -->|ack when minISR==2| P
    L -->|high watermark| C[Consumer reads committed tail]
    F3[Follower - Broker 4 - OUT of sync] -.->|slow, excluded from ISR| L
```

## 8. Request or Data Flow
1. Producer appends to leader (acks=all). Each ISR follower fetches and appends.
2. When the high watermark advances past the record on all ISR replicas, leader acks the producer.
3. Consumers read only ≤ high watermark — no reading unacknowledged tail.
4. Leader dies → controller elects an ISR replica → new leader serves; producers redirect (metadata refresh).
5. **No ISR available:** writes stop (or unclean election enables loss-tolerant mode).

## 9. Practical Example
**Time-series pipeline (assumptions):** RF=3, minISR=2, acks=all, 3 brokers across 2 AZs.
- A broker in AZ-A loses power. The partitions it led move to ISR replicas in AZ-B; producers pause ~seconds (leader-not-available retry), then continue. No messages lost; consumers see a tiny stall while metadata refresh happens.
- During the outage, ISR is 2 → writes still ack (minISR=2). If a *second* broker dies, ISR=1 < minISR=2 → writes are rejected with NOT_ENOUGH_REPLICAS rather than silently written single-copy — the system *tells you* it can't guarantee durability.

## 10. Scaling
- **Throughput:** reads scale with consumer groups; writes are leader-serialized per partition (more partitions = more write parallelism across brokers).
- **Storage:** RF multiplies storage linearly (3× for RF=3) — you also write/broadcast beyond 2/1, so choose RF per partition importance, not uniformly.
- **Reads on replicas:** leading consumers to read from followers is possible (rack-aware) but complicates consistency (stale reads) — default is leader-served reads.
- **Cross-AZ:** replicate across availability zones/racks for disaster resilience; adds inter-AZ bandwidth cost and latency.

## 11. Reliability and Failure Scenarios

| Failure | Happens | Detection | Recovery | Trade-off |
|---------|---------|-----------|----------|-----------|
| Leader crash | Failover to ISR member | Controller/health | Automatic election | small unavailability window |
| Follower lags past ISR | Shrunk ISR | ISR metric | Catch up → rejoin | durability drops |
| ISR < minISR | Writes rejected | NOT_ENOUGH_REPLICAS alerts | Restore replicas | availability |
| Pause (partition) | Both replicas unresponsive | No leader, no writes | Wait for quorum/recovery | availability vs loss |
| Unclean election allowed | Tail lost | Offset discontinuity | Rebuild from upstream | availability |
| Disk failure | Replica permanently lost | Disk alerts | Re-add + full replication | rebalance cost |

## 12. Consistency and Correctness
- **The durable write** is one acked by all ISR members: anything less (acks=1, or ISR=1) *can* be lost on failover. If you can't lose events, acks=all + minISR ≥ 2 is the operating point.
- **High watermark = read boundary:** consumers never see the unacknowledged tail, so no "read a record then it disappeared" surprise. This is a strong read-consistency property.
- **Failure is binary:** with settings tuned, outcomes are "safe failover" or "explicit unavailability" — the ambiguous, lossy middle is *unclean leader election*; leave it off unless availability trumps correctness.

## 13. Performance
- acks=all rounds-trips to replicas, adding latency (~an extra RTT) but no read cost.
- Followers consume cluster bandwidth (RF × message size); inter-AZ adds WAN cost.
- Very high write throughput: batch + compress, raise minISR only as durability truly demands, keep followers healthy (rebalance slowly, don't starve them).

## 14. Security
- Broker-to-broker mTLS and internal ACLs (replication traffic must be restricted, not accessible to clients).
- Only the controller/ops path may change RF or enable unclean election — treat as high-risk config with change control and review before flipping.

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| acks=1 | Low latency | Tail loss on leader crash | Non-critical, durable-elsewhere |
| acks=all, minISR=2 | No loss under 1 failure | +latency; rejects when replicas down | Ledger, orders, money-adjacent |
| MinISR=1 | Always writable | "Committed" may be single-copy | Best-effort streams |
| Unclean ON | Never leaderless | Lossy failover | Telemetry/transient data only |
| Unclean OFF (default) | Never lossy | Can be unavailable | Correctness matters |

## 16. Common Mistakes
- RF=3 with acks=1/minISR=1 — the SF architecture "has replication" but actually accepts single-copy writes.
- Forgetting `unclean.leader.election.enable` defaults and discovering the cluster chose loss.
- Treating consumer lag as the same risk as replica lag (ISR lag can cause write rejection, a different beast).
- Growing RF without monitoring inter-AZ bandwidth and replica-fetch latency.

## 17. HLD vs LLD Boundary
HLD: RF, ISR policy, minISR, ack model per stream, AZ/rack placement, unclean-election policy, failover time budgets. LLD: broker `server.properties` for replica lag tuning, controller election config, producer acks wires.

## 18. Interview Questions

### Beginner
- What is the ISR and why does leadership require it?
- Why can't consumers read past the high watermark?

### Intermediate
- acks=all vs acks=1 with RF=3: exactly which failure each survives?
- A follower falls out of ISR. What happens to writes and to failover risk while it's gone?

### Advanced
- Design RF/minISR/acks for a payments stream that must lose zero events across one full-AZ failure while staying writable in a 3-node 2-AZ cluster. Show the failure math.
- Toggle unclean leader election in a disaster scenario and describe exactly the trade-off you're accepting.
- How do you rebalance replication when a resized broker joins (data moves) without losing ack guarantees?

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- One leader + N−1 copying followers per partition; followers serve neither reads nor writes.
- ISR = in-sync replicas — only they may become leader and participate in acks=all.
- High watermark = the tail all ISR replicas hold = the only tail consumers may read.
- acks=all + minISR = the durability dial; unclean leader election = the loss/availability switch.
- RF=3 is standard but meaningless without matching acks and minISR.
- Example worth keeping: the 3-broker 2-AZ pipeline and the ISR < minISR rejection walkthrough.

### 30-Second Explanation

Replication gives copies; ISR decides safety. Writes ack when minISR confirms, failover promotes only ISR, and consumers read only the committed watermark — so no loss unless you explicitly allow unclean elections.

### Interview Traps

- Quoting "replication factor 3" as the durability answer while ignoring acks/minISR — durability is decided by who must ack, not how many copies exist.
- Treating consumer lag as the same risk as replica lag — ISR lag can reject writes, a different beast.
- Forgetting `unclean.leader.election.enable` defaults and discovering the cluster chose loss.

### Key Trade-Off

Replication trades write latency and 3× storage for durability and failover — and the real dial is who must ack (ISR/minISR), not the copy count: prefer availability (unclean on) or prefer durability (unclean off).

## 20. Related Concepts

### Prerequisites

- [[kafka-architecture|Kafka Architecture]]
- [[kafka-cluster|Kafka Cluster]]

### Commonly Used Together

- [[kafka-delivery-guarantees|Kafka Delivery Guarantees]]
- [[kafka-producers-consumers|Kafka Producers and Consumers]]
- [[kafka-ordering|Kafka Ordering]]
- [[failover|Failover]]
- [[database-replication|Database Replication]]

### Advanced Concepts

- [[cap-theorem|CAP Theorem]]
- [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]]

Related planned topics (not authored yet): quorum and epoch fencing, ISR lag internals, rack-aware replica placement, tail-loss semantics.

## 21. References
Apache Kafka replication and high-watermark docs. Verify exact ISR/lag semantics per broker version for interviews.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- What is the ISR, and why does leadership require it?
> The ISR (in-sync replicas) is the set of followers caught up with the leader within `replica.lag.time.max.ms`. Only ISR members may become leader and only they count toward acks=all — guaranteeing the new leader has every committed record, so a safe failover never loses or reorders the committed tail.

> [!question]- Why can't consumers read past the high watermark?
> The high watermark is the last offset held by all ISR replicas — the committed tail. Reading further would expose records that an unclean failover could lose, so consumers would see data that could later vanish. Restricting reads to the watermark is the strong read-consistency property: committed data never disappears.

> [!question]- With RF=3, exactly which failure does acks=all survive that acks=1 doesn't?
> acks=1 acks on the leader alone; if the leader crashes before its followers replicate the tail, those last records are lost. acks=all acks only when all in-sync replicas confirm, so a failover to an ISR member preserves every acknowledged record — it survives a leader crash mid-write, at the cost of an extra replication round-trip of latency.

> [!question]- A follower falls out of ISR. What happens to writes and to failover risk while it's gone?
> Writes: with acks=all and minISR satisfied by remaining replicas, writes continue; if ISR drops below minISR, writes are rejected with NOT_ENOUGH_REPLICAS — the system refuses to pretend single-copy durability. Failover risk: with replicas still in ISR, failover is safe; risk rises only if all remaining ISR replicas fail.

> [!question]- Design RF/minISR/acks for a payments stream that must lose zero events across one full AZ failure in a 3-node 2-AZ cluster, staying writable.
> RF=3 spread across both AZs, `min.insync.replicas=2`, `acks=all` with unclean election off. One full-AZ failure leaves at least one replica (ISR ≥ 1), but writeability needs ISR ≥ 2 — so this survives the AZ loss with *no data loss* by becoming unavailable for writes, or you accept a 4-node/3-AZ layout to stay writable with the math shown.

> [!question]- Toggle unclean leader election in a disaster scenario: what trade-off are you accepting?
> Turning it on lets the controller promote a follower even when it's out of sync — the cluster stays available (writes resume) but the new leader's log is missing the tail that the old leader had; those records are permanently lost and ordering may have gaps. Off (default) keeps the data but the partition can be unavailable until an ISR replica recovers.

> [!question]- Interview scenario: "We set RF=3, so we're fully durable." How do you respond?
> RF=3 alone doesn't guarantee durability. Durability is decided by the ack path: with acks=1 the leader can ack and crash before replicas copy the tail (loss); with minISR=1 an "acked" write can be a single copy. The durable operating point is acks=all + minISR ≥ 2 + RF ≥ 3 and unclean election off — walk those together.

> [!question]- Failure: ISR shrinks to 1 while minISR=2. What does the system do and why is that better than alternatives?
> Writes fail with NOT_ENOUGH_REPLICAS instead of being accepted as single-copy "committed". That trades availability for honesty — silently continuing would let a next failure lose everything. Recovery: restore a healthy follower so ISR ≥ minISR again, then writes resume by themselves.

> [!question]- How does replication cost scale, and how do you control it?
> Storage is RF × data size and replica fetching consumes cluster bandwidth (RF × message size), plus inter-AZ costs for cross-region placements. Control by choosing RF per partition importance (not uniformly 3 everywhere), keeping followers healthy so they don't fall behind, and batching/compressing before replication.

## 23. When Should I Use This?

### Use it when

- A topic's committed data must survive broker or AZ failures (RF=3 + minISR).
- You have ledger-adjacent streams where silent tail loss is unacceptable.
- Consumers must never read data that a failover could take back (watermark reads).
- You want automatic failover with no client-side leader logic.

### Avoid it when

- The stream is disposable telemetry — RF=1 or acks=0 is cheaper and fine.
- "Replication" is being used as a slogan without matching acks/minISR (you don't actually have it).
- You're not willing to pay the storage multiplier and failover latency for the copies.
- Unavailability is worse than loss — then unclean election is a deliberate, documented choice instead.

### What problem does it solve?

A single-copy partition is a single point of failure: a broker crash loses the data. The bottleneck is one machine's durability. Replication spreads copies (RF), ISR defines which copies are trustworthy enough to lead and ack, and the high watermark hides the uncommitted tail — so failover is safe, orderly, and lossless by default, with an explicit switch (unclean election) for availability-over-correctness.

### What problem does it NOT solve?

Durability without matching acks/minISR (copies alone don't protect the tail), exactly-once delivery to consumers (still at-least-once downstream), and read scaling on leaders by default — replicas don't serve reads unless you opt into stale-read fan-out.

## 24. Decision Connections

Decisions that go together with Kafka replication:

- [[kafka-cluster|Kafka Cluster]] — RF, broker count, and AZ layout are decided at cluster/topic design time.
- [[kafka-delivery-guarantees|Kafka Delivery Guarantees]] — acks/isolation are how replication durability becomes a delivery guarantee.
- [[kafka-producers-consumers|Kafka Producers and Consumers]] — producer durability budget and consumer read-consistency depend on acks/watermark.
- [[kafka-ordering|Kafka Ordering]] — ISR-based failover preserves per-partition order of the committed tail.
- [[failover|Failover]] — the general leader-failover machinery Kafka's election implements.
- [[database-replication|Database Replication]] — the same replica/duration reasoning applied to databases.
- [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] — watermark reads give committed-then-seen consistency, the strong end of the spectrum.

Decision tree:

```
How much durability for this topic?
    |
    +-- Disposable telemetry?             → RF=1, acks=0/1
    |
    +-- Survive broker crash, no loss?    → RF=3 + acks=all + minISR>=2
    |      +-- Survive full AZ failure?   → replicas across AZs
    |      +-- Must stay writable too?    → more nodes/AZs (math at design time)
    |
    +-- Must never be unavailable (loss ok)?
           → unclean leader election ON
           → [[kafka-delivery-guarantees|Kafka Delivery Guarantees]]
```