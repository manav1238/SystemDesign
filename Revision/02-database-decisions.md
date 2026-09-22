---
title: Database Decisions
status: active
tags:
  - hld
  - revision
  - databases
---

# Database Decisions

Decision tree for data-layer questions. Walk from the problem, not from the technology.

## 1. Which database type?

- Fixed schema, joins, transactions, relational integrity → [[sql-vs-nosql|SQL vs NoSQL]]
- Flexible schema, horizontal scale, distributed data → [[sql-vs-nosql|SQL vs NoSQL]] (NoSQL side)
- Unsure how to even frame the choice? Start from [[database-fundamentals|Database Fundamentals]]: data model + access pattern drive everything.

## 2. How do I make queries fast?

- Reads slow but writes fine → check indexing first, then caching → [[database-indexing|Database Indexing]] → [[caching|Caching]]
- Redundant/slow joins, heavy read workload → denormalize → [[normalization-vs-denormalization|Normalization vs Denormalization]]
- Hot row/column lookups → verify key design → [[database-keys|Database Keys]]
- Connections bottleneck under load → pool them → [[database-connection-pooling|Database Connection Pooling]]

## 3. How do I scale reads?

- Read-heavy, tolerate bounded staleness → replicas → [[database-replication|Database Replication]]
- Must read your own writes immediately → watch [[replication-lag|Replication Lag]], route risky reads to leader
- Zero downtime when the primary dies → automatic failover → [[failover|Failover]]

## 4. How do I scale writes / total data?

- One DB too big or one hotspot is fatal → shard → [[sharding|Sharding]] and choose → [[shard-key|Shard Key]] carefully, then [[sharding-strategies|Sharding Strategies]]
- Shard redistribution must be cheap on churn → [[consistent-hashing|Consistent Hashing]]
- Data fits one DB, just organized better → [[partitioning-vs-sharding|Partitioning vs Sharding]]

## 5. What consistency can I promise?

- Multi-node, strong ordering required → [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] (strong side)
- Must survive partitions at scale → you are choosing availability over strong consistency → [[cap-theorem|CAP Theorem]]
- Distributed writes must be atomic with an event → [[outbox-pattern|Outbox Pattern]] (see messaging)

## One-liner

```
Need read scale?    → [[database-replication|Database Replication]] → [[replication-lag|Replication Lag]] → [[failover|Failover]]
Need write scale?   → [[sharding|Sharding]]
Need atomic multi-row? → [[transactions-and-acid|Transactions and ACID]] (single node) or [[outbox-pattern|Outbox Pattern]] (with events)
```