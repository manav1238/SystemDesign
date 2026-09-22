---
title: Messaging Decisions
status: active
tags:
  - hld
  - revision
  - messaging
---

# Messaging Decisions

Decision tree for async / event-driven questions.

## 1. Should I go async at all?

- Bursty load, slow downstream, need decoupling → [[message-queue|Message Queue]]
- Must fan one event out to many independent consumers → [[publish-subscribe|Publish/Subscribe]]
- Services should react to domain events, not call each other → [[event-driven-architecture|Event-Driven Architecture]]
- Need hard ordering + replay + scale → Kafka family (below)

## 2. Queue vs topic vs Kafka?

- Simple work queue, one consumer group, strong guarantees → [[message-queue|Message Queue]]
- Broadcast to N consumers → [[publish-subscribe|Publish/Subscribe]]
- High throughput log replay + ordering + retention → [[kafka-architecture|Kafka Architecture]] with [[kafka-cluster|Kafka Cluster]]
- Producer/consumer mechanics → [[kafka-producers-consumers|Kafka Producers and Consumers]]; ordering scope → [[kafka-ordering|Kafka Ordering]]; durability dial → [[kafka-replication|Kafka Replication]]

## 3. What happens when processing is slow?

- Backlog grows on the consumer side → [[consumer-lag|Consumer Lag]]; add partitions/consumers or shed load
- Rebalances while scaling/failing → [[kafka-rebalancing|Kafka Rebalancing]]

## 4. What happens when things fail?

- Message lost, duplicated, or both → pick semantics → [[delivery-semantics|Delivery Semantics]]
  - At-least-once is the default: downstream must be idempotent
  - Exactly-once: add dedup — see also [[kafka-delivery-guarantees|Kafka Delivery Guarantees]]
- DB write + event must be atomic → [[outbox-pattern|Outbox Pattern]]

## 5. Retention and replay?

- Re-process historical events after bug/backfill → enable replay, keep retention → [[kafka-retention|Kafka Retention]]

## Decision tree

```
Async needed?
  Yes → queue (one consumer) → [[message-queue|Message Queue]]
        broadcast (many)    → [[publish-subscribe|Publish/Subscribe]]
        events as state     → [[event-driven-architecture|Event-Driven Architecture]]
        scale+replay+order  → [[kafka-architecture|Kafka Architecture]]
  Exactly-once needed? → [[delivery-semantics|Delivery Semantics]] + idempotent consumer
  DB+event atomic?     → [[outbox-pattern|Outbox Pattern]]
```