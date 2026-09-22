---
title: HLD Interview Checklist
status: active
tags:
  - hld
  - revision
  - interview
---

# HLD Interview Checklist

Run this skeleton on any design question. One-liners only; drill until it is automatic.

## Phase 1 — Requirements (2–3 min)

- Collect functional requirements; clarify scope: who, what, scale.
- Nail non-functional requirements: availability, latency, consistency, durability targets → [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]]
- If the interviewer does not define scale, propose it before estimating.

## Phase 2 — Estimate (2 min)

- DAU → QPS (active ratio × peak factor) → storage → bandwidth → [[capacity-estimation|Capacity Estimation]]
- State assumptions out loud; you will be judged on the method, not the numbers.

## Phase 3 — High-level design (5 min)

- Draw boxes: [[dns|DNS]] → [[cdn|CDN]] → [[load-balancing|Load Balancing]] + [[reverse-proxy|Reverse Proxy]] → app services → [[caching|Caching]] → DB.
- Decide stateless vs stateful for each layer → [[stateless-vs-stateful-services|Stateless vs Stateful Services]].

## Phase 4 — Deep dive (driven by goal)

- Read-heavy? → [[database-replication|Database Replication]], [[caching|Caching]], [[cdn|CDN]]
- Write-heavy / huge data? → [[sharding|Sharding]], [[shard-key|Shard Key]], [[consistent-hashing|Consistent Hashing]]
- Async / decouple? → [[message-queue|Message Queue]], [[publish-subscribe|Publish/Subscribe]], [[event-driven-architecture|Event-Driven Architecture]], [[outbox-pattern|Outbox Pattern]]
- Failure safety? → [[circuit-breaker|Circuit Breaker]], [[retry-and-timeout|Retry and Timeout]], [[rate-limiter|Rate Limiter]], [[failover|Failover]]
- Consistency? → [[cap-theorem|CAP Theorem]], [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]], [[kafka-ordering|Kafka Ordering]]

## Phase 5 — Trade-offs and follow-ups

- For every choice, state the cost: e.g. caching → forgiveness of ETL/API; asynchronous ordering; sharding → query complexity.
- Security paragraph — auth, TLS, input validation → [[authentication-vs-authorization|Authentication vs Authorization]], [[encryption-and-keys|Encryption and Keys]], [[web-vulnerabilities|Web Vulnerabilities]]
- Observability paragraph — golden signals + tracing + SLO → [[golden-signals|Golden Signals]], [[distributed-tracing|Distributed Tracing]], [[sli-slo-sla|SLI / SLO / SLA]]

## Failure-mode drill (rapid fire)

- "EC2 instance dies" → load balancer drains it, autoscaler replaces, stateless services are fine → [[load-balancing|Load Balancing]]
- "DB primary dies" → failover → [[failover|Failover]]
- "Read replica lags" → route read-your-writes to primary → [[replication-lag|Replication Lag]]
- "Cache thrashes" → stampede/penetration/avalanche → [[caching|Caching]]
- "One shard overloaded" → bad shard key / hot key → [[shard-key|Shard Key]]
- "Consumer falls behind" → scale partitions/consumers → [[consumer-lag|Consumer Lag]]
- "Downstream dies" → circuit opens → [[circuit-breaker|Circuit Breaker]]

## Flow

Quick-fire one-liners → [[00-index|Master Index]] for syllabus, or [[01-rapid-revision|Rapid Revision]] to warm up.