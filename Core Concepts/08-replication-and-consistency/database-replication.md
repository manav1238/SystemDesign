---
title: Database Replication
category: Database
priority: must-know
status: learning
difficulty: medium
interview_ready: false
tags:
  - hld
  - database
  - replication
---

# Database Replication

## 1. One-Line Definition
Replication keeps multiple synchronized copies (replicas) of the same database at different nodes, used for high availability, disaster recovery, and scaling reads.

## 2. Why Do We Need It?
You cannot have availability without copies: a single database is a single point of failure. Replication gives you read scale (many copies serve reads), failover (another copy becomes primary), durability (a copy survives the primary's disk loss), and geo distribution (copies near users).

## 3. Simple Intuition
A manager keeps a personal notebook (primary); assistants get photocopies (replicas). Orders (writes) go to the manager; anyone can read a photocopy. If the manager leaves, an assistant's notebook is promoted — but note: anything written in the last few minutes may not be in the photocopies yet (replication lag).

## 4. What Happens Without It?
One database, one point of failure: it dies → everything dies. Reads all pile on the primary (bottleneck), a disk crash destroys acknowledged data, and a region outage takes every user down. You are one `apt upgrade` away from an outage.

## 5. Core Idea
- **Leader-follower (primary-replica):** single leader accepts writes, ships change stream to followers.
  - **Sync replication:** only ack a write after ≥1 follower has it → strong-ish, but slow + unavailable if followers drop.
  - **Async replication:** ack immediately, followers catch up → fast; on leader death, unsent writes are lost (RPO > 0).
  - **Semi-sync:** ack after one follower confirms → durability + speed (common production trade).
- **Multi-leader / leaderless:** multiple write nodes (multi-master, Cassandra/Riak style) → write-anywhere latency, at the price of conflict resolution.
- **Read scaling via followers:** reads hit followers; critical reads go to leader or quorum.

**The one question that decides the mode:** *what RPO (data loss) do you accept on leader failure?* Sync=0 but requires a healthy follower; async=tiny (ms-seconds).

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Leader / primary | The only writer |
| Follower / replica / slave | Read copy shipping from leader |
| Change stream / binlog / WAL shipping | The leader-to-follower data channel |
| Sync / async / semi-sync | Durability vs speed framing |
| Replication lag | Time follower lags behind leader |
| Promotion / failover | Making a follower the new leader |
| RPO | Max data loss window on failure |
| Multi-leader / leaderless | Multiple writers, conflict-aware |

## 7. Basic Architecture

```mermaid
flowchart LR
    App --> L[Leader (writes)]
    L -. async/sync .-> F1[(Replica 1)]
    L -. async/sync .-> F2[(Replica 2)]
    Rd[Reads] --> F1
    Rd[Reads] --> F2
    App --> Rd
```

## 8. Request or Data Flow
1. Write → leader (durably acked per sync mode).
2. Leader appends the change to its log/stream; followers apply it.
3. Reads → followers (may be slightly stale); critical reads pinned to leader or quorum.
4. Leader death → a follower is promoted; replicas re-point; app reconnect target flips.

## 9. Practical Example
**Profile service (assumptions):** 50k reads/sec, 2k writes/sec — reads dominant, staleness-per-profile acceptable seconds.
- 1 leader + 4 followers with **semi-sync** (durability, low write hit).
- Reads round-robin across followers; writes → leader.
- Leader dies → follower promoted (RPO ~0 with semi-sync), fleet re-points.

## 10. Scaling
- **Read scaling:** add followers for read QPS (up to a point — each followers duplicates storage + lag).
- **Write scaling:** replication does NOT scale writes — only one leader writes. For write scale → shard/partition.
- **Storage:** 1 copy = ×(1+replicas). Watch disk.
- **Global:** replicas in a region near users; cross-region async copy for DR.

## 11. Reliability and Failure Scenarios

| Failure | What Happens | Detection | Recovery | Trade-off |
|---------|--------------|-----------|----------|-----------|
| Leader death | Writes fail | Health + lag alerts | Promote a follower | async = loss window |
| Follower death | Reads degrade | Lag alerts | Spare/rebuild from snapshot | reads miss those nodes |
| Replication lag | Stale reads | Lag metric | Route reads away | freshness vs load |
| Split brain | Two leaders | Heartbeat/quorum | Automated fencing/quorum | availability vs complexity |
| Disk corruption | Copies poison | Checksums/checks | Restore that copy | transparency cost |

## 12. Consistency and Correctness
Replication creates **eventual consistency** by default (followers lag). Correctness levers:
- Critical reads to leader / synchronous path.
- Read-your-writes patterns (session reads go to leader until ack flows) — see consistency-models.md.
- Monotonic reads: read from one follower per session to avoid "went back in time".
- Conflicts at multi-leader: LWW / version vectors — never hand-wave.

## 13. Performance
- Async writes are cheap (agg ack); sync/semi-sync adds follower-ack latency.
- Add read capacity: followers ≈ linear read scaling.
- Leader write bottleneck remains — monitor write QPS vs follower catch-up; heavy analytic queries must hit followers, not the leader.

## 14. Security
- Replication channel must be **encrypted** + authenticated (TLS between nodes).
- Replicas carry full data → at-rest encryption + same access controls as leader.
- Never expose follower ports publicly; all DB admins use least-privilege roles including on replicas (they're full copies!).

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| Async replication | Fast, available | Possible data loss window | Non-critical, feeds |
| Sync replication | Zero loss | Slow writes, needs a live follower | Money, critical |
| Semi-sync | Durability + speed | Slight write cost | Most production |
| Multi-leader | Local write latency | Conflict resolution burden | Multi-region writes, offline sync |
| Leaderless (quorum) | Write resilience | Read/write amplification | Cassandra-style workloads |

## 16. Common Mistakes
- Replicas used for the SAME reads as primary, then "read-your-writes" breaks — design the read-routing rule.
- Async replication sold as "no data loss" when RPO > 0 in reality.
- Not automating promotion — a dead leader with a human at 3am is an RTO of hours.
- Believing replication solves *write* scaling (it doesn't — see sharding).
- Ignoring lag alerts until an incident reveals reads 5 minutes stale.

## 17. HLD vs LLD Boundary
HLD: replication topology + mode + RPO/RTO policy + read-routing rules. LLD: the specific client failover logic, connection re-pointing or auto-discovery code.

## 18. Interview Questions

### Beginner
- Why do we replicate a database? Name all three reasons.
- What is synchronous vs asynchronous replication?

### Intermediate
- Your writes are the bottleneck: does replication help? Why or why not?
- Leader dies with async replication — what is lost and how do you contain it?

### Advanced
- Design a global product that needs same-region write + zero-loss failover. Walk the topology.
- How do you detect and survive split-brain after a partition heals?

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- Copies → availability + DR + read scale.
- One leader writes; the replication mode is your RPO story.
- Sync = durability; async = speed; semi-sync = production balance.
- Reads go to followers; critical reads pinned to leader/quorum.
- Replication ≠ writes; sharding handles write scale.
- The decision driver: what RPO do you accept on leader failure?

### 30-Second Explanation

Leader + async/semi-sync followers: reads to followers, writes to leader, automated promotion + quorum fencing, monitor lag.

### Interview Traps

- "Replicas make us highly consistent" — they make reads *available* and *eventually* consistent; consistency is a routing + mode decision.
- Using replicas for the same reads as primary, then read-your-writes breaks.
- Selling async as "no data loss" when RPO > 0 in reality.
- Not automating promotion (human at 3am = RTO of hours).
- Believing replication scales writes (it doesn't — see sharding).

### Key Trade-Off

Replication trades a small write cost (and possible async loss window) for availability, durability, and cheap read scaling; write and storage scaling still require sharding, which replication cannot provide.

## 20. Related Concepts

### Prerequisites

- [[database-fundamentals|Database Fundamentals]]
- [[database-connection-pooling|Database Connection Pooling]]

### Commonly Used Together

- [[replication-lag|Replication Lag]]
- [[failover|Failover]]
- [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]]

### Alternatives

- [[sharding|Sharding]]

### Advanced Concepts

- [[cap-theorem|CAP Theorem]]
- [[rpo-rto|RPO and RTO]]
- [[standby-models|Standby Models]]
- [[disaster-recovery|Disaster Recovery]]

Related planned topics (not authored yet): quorum.

## 21. References
PostgreSQL/MySQL replication docs; Kleppmann ch. 5. Verify with current engine docs.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- Why do we replicate a database? Name the three reasons.
> 1) **Availability/high availability** — a copy survives the primary's death; 2) **disaster recovery** — a copy at another site survives disk/region loss; 3) **read scaling** — many replicas serve reads. Replication does not scale writes — only one leader accepts them.

> [!question]- What is synchronous vs asynchronous replication?
> **Sync:** only ack a write after ≥1 follower has it → near-zero loss (RPO 0) but slow, and unavailable if followers drop. **Async:** ack immediately, followers catch up → fast, but on leader death unsent writes are lost (RPO > 0). **Semi-sync** is the middle: ack after one follower confirms.

> [!question]- Design decision: profile service with 50k reads/sec, 2k writes/sec. What topology?
> 1 leader + 4 followers with **semi-sync** (durability + low write hit). Reads round-robin across followers; writes → leader; critical reads pinned to leader/quorum. Leader dies → automated promotion + re-point (RPO ~0 with semi-sync).

> [!question]- Trade-off: async vs sync replication — which do you pick and why?
> Sync gives durability at the cost of write latency and availability (if no follower is healthy, writes stop). Async gives speed and availability but a loss window. Pick by RPO: money/critical → sync or semi-sync; tolerant data → async. Semi-sync is the common production trade.

> [!question]- Failure scenario: leader dies with async replication. What is lost and how do you contain it?
> Every write acked but not yet shipped to followers is lost (RPO = that tail). Contain it by promotion that only loses the unsynced tail, capturing a known recovery point, then backfilling the new leader from replicas that have that data and reconciling.

> [!question]- Interview scenario: "Replicas make us highly consistent." How do you respond?
> Replicas make reads highly *available* and *eventually* consistent; consistency is a routing-plus-mode decision. Read-your-writes and monotonicity need pinned sessions/leader reads; critical reads go to the leader/quorum. Replication is durability + read scale, not a consistency guarantee.

> [!question]- Interview scenario: can replication fix write bottlenecks?
> No — one leader still receives every write, so write QPS stays capped at one node. Reads scale with followers, but writes/storage only scale with sharding/partitioning. Know the boundary: replication for reads/HA, sharding for writes/storage.

## 23. When Should I Use This?

### Use it when

- You need availability while a primary node fails.
- Reads dominate and one primary can't serve them (followers scale reads).
- You need crash/disk-loss safety or a cross-region copy for DR.
- You can tolerate (or control) replication lag on most read paths.

### Avoid it when

- Writes are the bottleneck — replication can't scale writes; shard instead.
- Every read must see the latest write (followers lag; use leader/quorum reads).
- Storage cost of ×(1+replicas) copies is unacceptable.
- You expect replicas to make reads strongly consistent without routing care.

### What problem does it solve?

Problem: one database is a single point of failure and a read bottleneck. Bottleneck: no copies means a node death is an outage and the primary absorbs all reads. Solution: replication keeps synchronized copies that serve reads, enable failover, survive disk loss, and bring data near users.

### What problem does it NOT solve?

It doesn't scale writes — the leader is still one writer; it doesn't guarantee fresh reads (followers lag — you must route critical reads to leader/quorum); and it doesn't fix storage scale (copies multiply, not divide, the working set). Those are sharding's job.

## 24. Decision Connections

Decisions that go together with Database Replication:

- [[database-fundamentals|Database Fundamentals]] — replication is the layer above the single node.
- [[database-connection-pooling|Database Connection Pooling]] — read replicas each need their own pool.
- [[replication-lag|Replication Lag]] — the freshness cost that decides read routing.
- [[failover|Failover]] — promotion + quorum + fencing on top of replicas.
- [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] — what readers actually get.
- [[rpo-rto|RPO and RTO]] — the numbers your sync/async mode commits to.
- [[standby-models|Standby Models]] — the standby/topology variants on the same idea.
- [[sharding|Sharding]] — the complementary move for writes/storage.

Decision tree:

```
Need copies of the data?
    |
    +-- Reads dominate, data fits?
    |      → [[database-replication|Database Replication]] (read replicas)
    |
    +-- Zero-loss durability on failure?
    |      → sync/semi-sync ([[rpo-rto|RPO and RTO]])
    |
    +-- Node death must not break writes?
    |      → [[failover|Failover]] (promotion + quorum + fencing)
    |
    +-- Reads must be fresh after a local write?
    |      → pin session / leader reads ([[replication-lag|Replication Lag]])
    |
    +-- Writes or storage exceed one node?
    |      → [[sharding|Sharding]] (replication doesn't scale writes)
```