---
title: Rapid Revision
status: active
tags:
  - hld
  - revision
---

# Rapid Revision

One-liner recall for every authored concept. Say the answer aloud, then click through if needed. Cover one area per session; mark concepts you miss, then revisit the full note.

## Fundamentals and Estimation (10)

- **System Design Fundamentals** — define functional vs non-functional requirements and constraints, then estimate.
- **Functional vs Non-Functional Requirements** — functional = what it does; non-functional = SLAs/constraints (perf, scale, security).
- **Scalability** — capacity to grow (load or data) without redesign.
- **Availability** — % of time the system serves; driven by redundancy and failure handling.
- **Reliability** — works when expected despite failures; availability is the measured outcome.
- **Latency vs Throughput** — latency = time per request; throughput = requests/unit time; trade them off with parallelism/queues.
- **Bottleneck Identification** — find the slowest link (CPU, DB, network, lock) before scaling.
- **Stateless vs Stateful Services** — stateless scales horizontally trivially; stateful needs affinity/replication/shards.
- **Horizontal vs Vertical Scaling** — add machines vs make one bigger; horizontal wins at scale.
- **Capacity Estimation** — QPS, storage, bandwidth from DAU/MAU + peak factor; do it early and small.

## Networking and Traffic (4)

- **DNS** — global name→IP; caching at every level; the first lookup you must mention.
- **HTTP and HTTPS** — request/response, methods, status, caching headers; TLS on top for security.
- **Load Balancing** — distribute traffic across servers; L4 vs L7; health checks + draining.
- **Reverse Proxy** — single front door: TLS, caching, compression, shielding backends.

## Caching and CDN (2)

- **Caching** — store hot results; cache-aside vs write-through/back; mind invalidation, stampede, penetration, avalanche.
- **CDN** — static content at edge servers near users; geo-routing, TTL, origin pull/push.

## Databases (7)

- **Database Fundamentals** — data models, schemas, storage engines; choose fit for access pattern.
- **SQL vs NoSQL** — SQL: schema, joins, ACID; NoSQL: flexible schema, horizontal scale; pick per workload.
- **Normalization vs Denormalization** — normalize for integrity; denormalize for read perf; document the trade-off.
- **Database Keys** — PK uniqueness, candidate/composite/foreign; keys drive lookups and sharding.
- **Database Indexing** — B-tree/LSM make reads fast at write cost; selective, covering indexes.
- **Transactions and ACID** — atomicity, consistency, isolation, durability; too expensive at scale → eventual consistency.
- **Database Connection Pooling** — reuse connections cap overhead; size = threads, not requests.

## Replication and Consistency (5)

- **Database Replication** — leader/read replicas for read scale + failover; sync vs async.
- **Replication Lag** — async replica lag breaks read-your-writes; mitigate with routing/leases/bounded staleness.
- **Failover** — promote replica/slave on leader loss; automate with health checks + leader election.
- **Strong vs Eventual Consistency** — strong = linearizable costs latency; eventual = fast but stale reads briefly.
- **CAP Theorem** — you pick two of consistency/availability/partition tolerance; network partitions happen, so you choose.

## Sharding (5)

- **Sharding** — split data across databases by a key when one DB is not enough.
- **Shard Key** — pick one that spreads load evenly AND matches queries; the hardest decision.
- **Partitioning vs Sharding** — partitioning = logical split inside a DB; sharding = across machines.
- **Sharding Strategies** — range (skew) vs hash (even, no range scans) vs directory (flexible, extra hop).
- **Consistent Hashing** — hash ring minimises rebalancing on node add/remove; virtual nodes smooth the distribution.

## Messaging (6)

- **Message Queue** — decouple producers/consumers, smooth bursts; adds latency + ordering constraints.
- **Publish/Subscribe** — one event to many consumers; fan-out; still a topic, not a queue.
- **Event-Driven Architecture** — services react to events instead of calling each other; stronger decoupling, harder reasoning.
- **Delivery Semantics** — at-most (loss), at-least (dup, need idempotency), exactly-once (dedup + ordering constraints).
- **Outbox Pattern** — write event with the DB transaction so queue + DB never diverge.
- **Consumer Lag** — the alerting canary for backend throughput; scale partitions + consumers when it grows.

## Kafka (8)

- **Kafka Architecture** — distributed commit log: topics → partitions → brokers; retain + replay.
- **Kafka Cluster** — brokers + ZK/KRaft coordination; partitions spread across brokers.
- **Kafka Producers and Consumers** — producers pick partitions by key; consumers committed offsets in a group.
- **Kafka Replication** — ISR set; min ISR + acks tune the durability/latency dial.
- **Kafka Ordering** — per-partition ordering only; your ordering scope = your keying.
- **Kafka Rebalancing** — partition reassignment on join/leave; the least predictable pause; tune session timeouts.
- **Kafka Retention** — time/size-based deletion or log compaction; replay timelines depend on it.
- **Kafka Delivery Guarantees** — acks, idempotent producer, transactions → exactly-once; costs throughput.

## Reliability (6)

- **Retry and Timeout** — bounded retries + exponential backoff/jitter; never retry non-idempotent blindly.
- **Circuit Breaker** — fail fast to protect a sick dependency; avoid retry storms.
- **Rate Limiter** — admission control: token bucket/leaky bucket; distinguish client quotas from overload.
- **Disaster Recovery** — backup/restore, replication, and a runbook; test it.
- **RPO and RTO** — RPO = data you can lose; RTO = time to recover; targets drive the standby choice.
- **Standby Models** — active-passive (cheap, slower failover) vs active-active (fast, harder consistency).

## Observability (4)

- **Observability** — logs, metrics, traces; the three pillars of "why is it failing".
- **Golden Signals** — latency, traffic, errors, saturation; the universal health gauge.
- **Distributed Tracing** — trace/correlation IDs across services to find the slow hop.
- **SLI / SLO / SLA** — measure → target → contract; the error budget is your alerting guide.

## Security (4)

- **Authentication vs Authorization** — prove who you are, then what you may do.
- **OAuth 2.0 / OIDC / JWT** — delegate identity; issuer-signed tokens; keep expiry + scope tight.
- **Encryption and Keys** — TLS in transit, envelope encryption at rest; keys under KMS.
- **Web Vulnerabilities** — injection, XSS, CSRF, SSRF; validate + sanitize at trust boundaries.