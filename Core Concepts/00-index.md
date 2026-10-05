---
title: HLD Core Concepts — Master Index
status: WIP
generated: 2026-09-19
---

# HLD Core Concepts — Master Index

Comprehensive High-Level Design (HLD) / system-design knowledge base.

- **Status Legend:** `Learning` · `Revised` · `Interview ready`
- **Priority Legend:** `Must Know` · `Important` · `Advanced` (mirrors each file's frontmatter `priority`)
- Every authored file follows the shared 24-section template (see `00-obsidian-readability.md`): core learning 1–18, Interview Memory Summary 19, Related Concepts 20, References 21, **Active Recall 22**, When Should I Use This 23, Decision Connections 24.
- Revision hub for exam-style recall: `../Revision/` (decision frameworks + interview checklist).

---

## Learning Path (Recommended Order)

1. **Foundations first** — `01-hld-fundamentals` → `02-estimation` → `03-networking`
2. **Traffic layer** — `04-api-design` → `05-load-balancing-and-traffic` → `06-caching-and-cdn`
3. **Data layer** — `07-databases` → `08-replication-and-consistency` → `09-sharding-and-partitioning`
4. **Async layer** — `10-messaging-and-event-driven` → `11-kafka`
5. **Hardening** — `12-reliability-and-resilience` → `14-observability` → `15-security`
6. **Platform** — `13-deployment-and-infrastructure` → `16-storage` → `17-search-and-data-processing`
7. **Deep distributed systems** — `18-distributed-systems` → `19-architecture-patterns` → `20-multi-region-systems` → `21-advanced-concepts`

**Prerequisites by track:** Networking < API Design < Load Balancing; Databases < Replication < Sharding; Messaging < Kafka; Fundamentals < everything else.
---

## 01 — HLD Fundamentals [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Asynchronous Processing | [asynchronous-processing.md](01-hld-fundamentals/asynchronous-processing.md) | Learning | Must Know | Easy |
| Availability | [availability.md](01-hld-fundamentals/availability.md) | Learning | Must Know | Easy |
| Backward Compatibility | [backward-compatibility.md](01-hld-fundamentals/backward-compatibility.md) | Learning | Important | Easy |
| Bottleneck Identification | [bottleneck-identification.md](01-hld-fundamentals/bottleneck-identification.md) | Learning | Must Know | Medium |
| Consistency | [consistency.md](01-hld-fundamentals/consistency.md) | Learning | Must Know | Medium |
| Control Plane vs Data Plane | [control-plane-vs-data-plane.md](01-hld-fundamentals/control-plane-vs-data-plane.md) | Learning | Advanced | Medium |
| Distributed Systems | [distributed-systems.md](01-hld-fundamentals/distributed-systems.md) | Learning | Advanced | Medium |
| Durability | [durability.md](01-hld-fundamentals/durability.md) | Learning | Must Know | Easy |
| Extensibility | [extensibility.md](01-hld-fundamentals/extensibility.md) | Learning | Important | Easy |
| Fault Tolerance | [fault-tolerance.md](01-hld-fundamentals/fault-tolerance.md) | Learning | Must Know | Medium |
| Functional vs Non-Functional Requirements | [functional-vs-non-functional-requirements.md](01-hld-fundamentals/functional-vs-non-functional-requirements.md) | Learning | Must Know | Easy |
| High Cohesion | [high-cohesion.md](01-hld-fundamentals/high-cohesion.md) | Learning | Important | Easy |
| Horizontal vs Vertical Scaling | [horizontal-vs-vertical-scaling.md](01-hld-fundamentals/horizontal-vs-vertical-scaling.md) | Learning | Must Know | Easy |
| Latency vs Throughput | [latency-vs-throughput.md](01-hld-fundamentals/latency-vs-throughput.md) | Learning | Must Know | Easy |
| Loose Coupling | [loose-coupling.md](01-hld-fundamentals/loose-coupling.md) | Learning | Important | Easy |
| Maintainability | [maintainability.md](01-hld-fundamentals/maintainability.md) | Learning | Important | Easy |
| Performance | [performance.md](01-hld-fundamentals/performance.md) | Learning | Must Know | Medium |
| Redundancy | [redundancy.md](01-hld-fundamentals/redundancy.md) | Learning | Must Know | Easy |
| Reliability | [reliability.md](01-hld-fundamentals/reliability.md) | Learning | Must Know | Medium |
| Resilience | [resilience.md](01-hld-fundamentals/resilience.md) | Learning | Must Know | Medium |
| Scalability | [scalability.md](01-hld-fundamentals/scalability.md) | Learning | Must Know | Easy |
| Service-Oriented Architecture | [service-oriented-architecture.md](01-hld-fundamentals/service-oriented-architecture.md) | Learning | Advanced | Medium |
| Shared-Nothing Architecture | [shared-nothing-architecture.md](01-hld-fundamentals/shared-nothing-architecture.md) | Learning | Advanced | Medium |
| Single Point of Failure | [single-point-of-failure.md](01-hld-fundamentals/single-point-of-failure.md) | Learning | Must Know | Easy |
| Stateless vs Stateful Services | [stateless-vs-stateful-services.md](01-hld-fundamentals/stateless-vs-stateful-services.md) | Learning | Must Know | Medium |
| Synchronous Processing | [synchronous-processing.md](01-hld-fundamentals/synchronous-processing.md) | Learning | Important | Easy |
| System Design Fundamentals | [system-design-fundamentals.md](01-hld-fundamentals/system-design-fundamentals.md) | Learning | Must Know | Easy |
| Trade-Off Analysis | [trade-off-analysis.md](01-hld-fundamentals/trade-off-analysis.md) | Learning | Must Know | Medium |
---
## 02 — Estimation [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Cache Size Estimation | [cache-size-estimation.md](02-estimation/cache-size-estimation.md) | Learning | Must Know | Medium |
| Capacity Estimation | [capacity-estimation.md](02-estimation/capacity-estimation.md) | Learning | Must Know | Medium |
| Cost Estimation | [cost-estimation.md](02-estimation/cost-estimation.md) | Learning | Advanced | Medium |
| DAU and MAU | [dau-mau.md](02-estimation/dau-mau.md) | Learning | Must Know | Easy |
| Latency Budget | [latency-budget.md](02-estimation/latency-budget.md) | Learning | Advanced | Medium |
| Memory Estimation | [memory-estimation.md](02-estimation/memory-estimation.md) | Learning | Important | Easy |
| Read/Write Ratio | [read-write-ratio.md](02-estimation/read-write-ratio.md) | Learning | Must Know | Easy |
| Replication Overhead | [replication-overhead.md](02-estimation/replication-overhead.md) | Learning | Important | Medium |
| Server Capacity | [server-capacity.md](02-estimation/server-capacity.md) | Learning | Important | Medium |
---
## 03 — Networking [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Client-Server Model | [client-server-model.md](03-networking/client-server-model.md) | Learning | Must Know | Easy |
| Connection Pooling | [connection-pooling.md](03-networking/connection-pooling.md) | Learning | Must Know | Medium |
| DNS | [dns.md](03-networking/dns.md) | Learning | Must Know | Easy |
| Firewall | [firewall.md](03-networking/firewall.md) | Learning | Important | Easy |
| Forward Proxy | [forward-proxy.md](03-networking/forward-proxy.md) | Learning | Important | Easy |
| HTTP and HTTPS | [http-and-https.md](03-networking/http-and-https.md) | Learning | Must Know | Medium |
| HTTP Cookies | [http-cookies.md](03-networking/http-cookies.md) | Learning | Important | Easy |
| IP Address (IPv4 vs IPv6) | [ip-address.md](03-networking/ip-address.md) | Learning | Important | Easy |
| NAT | [nat.md](03-networking/nat.md) | Learning | Advanced | Medium |
| Network Latency / Bandwidth / Packet Loss | [network-latency.md](03-networking/network-latency.md) | Learning | Must Know | Medium |
| Network Partition | [network-partition.md](03-networking/network-partition.md) | Learning | Must Know | Medium |
| Ports | [ports.md](03-networking/ports.md) | Learning | Important | Easy |
| SSE / Long Polling / Short Polling | [server-sent-events.md](03-networking/server-sent-events.md) | Learning | Important | Easy |
| TCP | [tcp.md](03-networking/tcp.md) | Learning | Important | Medium |
| UDP | [udp.md](03-networking/udp.md) | Learning | Important | Easy |
| VPC and Subnets | [vpc-and-subnets.md](03-networking/vpc-and-subnets.md) | Learning | Advanced | Medium |
| WebSockets | [websockets.md](03-networking/websockets.md) | Learning | Important | Medium |
---
## 04 — API Design [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| API Composition | [api-composition.md](04-api-design/api-composition.md) | Learning | Important | Medium |
| API Design Principles | [api-design-principles.md](04-api-design/api-design-principles.md) | Learning | Must Know | Easy |
| API Gateway | [api-gateway.md](04-api-design/api-gateway.md) | Learning | Must Know | Medium |
| API Timeouts | [api-timeouts.md](04-api-design/api-timeouts.md) | Learning | Important | Easy |
| API Versioning | [api-versioning.md](04-api-design/api-versioning.md) | Learning | Must Know | Easy |
| Backend for Frontend | [backend-for-frontend.md](04-api-design/backend-for-frontend.md) | Learning | Important | Easy |
| Bulk and Long-Running APIs | [bulk-and-long-running-apis.md](04-api-design/bulk-and-long-running-apis.md) | Learning | Advanced | Medium |
| Contract-First Design | [contract-first-design.md](04-api-design/contract-first-design.md) | Learning | Advanced | Medium |
| Error Handling | [error-handling.md](04-api-design/error-handling.md) | Learning | Must Know | Easy |
| Filtering / Sorting / Searching | [filtering-sorting-searching.md](04-api-design/filtering-sorting-searching.md) | Learning | Important | Easy |
| Idempotency | [idempotency.md](04-api-design/idempotency.md) | Learning | Must Know | Medium |
| Pagination | [pagination.md](04-api-design/pagination.md) | Learning | Must Know | Easy |
| Request Deduplication | [request-deduplication.md](04-api-design/request-deduplication.md) | Learning | Important | Medium |
| REST | [rest.md](04-api-design/rest.md) | Learning | Must Know | Easy |
| RPC / gRPC / GraphQL | [rpc-grpc-graphql.md](04-api-design/rpc-grpc-graphql.md) | Learning | Important | Medium |
| Session Management | [session-management.md](04-api-design/session-management.md) | Learning | Must Know | Medium |
| Webhooks | [webhooks.md](04-api-design/webhooks.md) | Learning | Important | Easy |
---
## 05 — Load Balancing and Traffic [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Backpressure | [backpressure.md](05-load-balancing-and-traffic/backpressure.md) | Learning | Must Know | Medium |
| Client-Side vs Server-Side LB | [client-side-vs-server-side-lb.md](05-load-balancing-and-traffic/client-side-vs-server-side-lb.md) | Learning | Important | Easy |
| Connection Draining | [connection-draining.md](05-load-balancing-and-traffic/connection-draining.md) | Learning | Important | Easy |
| Consistent Hashing Load Balancing | [consistent-hashing-load-balancing.md](05-load-balancing-and-traffic/consistent-hashing-load-balancing.md) | Learning | Advanced | Medium |
| DNS Load Balancing | [dns-load-balancing.md](05-load-balancing-and-traffic/dns-load-balancing.md) | Learning | Important | Medium |
| Global Load Balancing | [global-load-balancing.md](05-load-balancing-and-traffic/global-load-balancing.md) | Learning | Advanced | Medium |
| Graceful Degradation | [graceful-degradation.md](05-load-balancing-and-traffic/graceful-degradation.md) | Learning | Important | Easy |
| Health Checks (Active/Passive) | [health-checks.md](05-load-balancing-and-traffic/health-checks.md) | Learning | Must Know | Easy |
| Hotspot Handling | [hotspot-handling.md](05-load-balancing-and-traffic/hotspot-handling.md) | Learning | Advanced | Medium |
| Load Balancer Failover | [load-balancer-failover.md](05-load-balancing-and-traffic/load-balancer-failover.md) | Learning | Must Know | Medium |
| Load Balancing | [load-balancing.md](05-load-balancing-and-traffic/load-balancing.md) | Learning | Must Know | Medium |
| Overload Protection | [overload-protection.md](05-load-balancing-and-traffic/overload-protection.md) | Learning | Advanced | Medium |
| Reverse Proxy | [reverse-proxy.md](05-load-balancing-and-traffic/reverse-proxy.md) | Learning | Important | Medium |
| Sticky Sessions | [sticky-sessions.md](05-load-balancing-and-traffic/sticky-sessions.md) | Learning | Must Know | Medium |
---
## 06 — Caching and CDN [`Must Know · Important`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Negative Caching / Warming / Versioning | [cache-warming.md](06-caching-and-cdn/cache-warming.md) | Learning | Important | Medium |
| Caching | [caching.md](06-caching-and-cdn/caching.md) | Learning | Must Know | Medium |
| CDN | [cdn.md](06-caching-and-cdn/cdn.md) | Learning | Important | Medium |
| Browser / HTTP Caching | [http-caching.md](06-caching-and-cdn/http-caching.md) | Learning | Important | Easy |
| Memcached | [memcached.md](06-caching-and-cdn/memcached.md) | Learning | Important | Easy |
| Redis | [redis.md](06-caching-and-cdn/redis.md) | Learning | Must Know | Medium |
---
## 07 — Databases [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| BASE | [base.md](07-databases/base.md) | Learning | Must Know | Medium |
| B-Tree / LSM-Tree / Hash Index | [btree-lsm-hash-index.md](07-databases/btree-lsm-hash-index.md) | Learning | Important | Medium |
| Data Model Types (Doc / KV / Wide / Graph / TS) | [data-model-types.md](07-databases/data-model-types.md) | Learning | Important | Medium |
| Database Connection Pooling | [database-connection-pooling.md](07-databases/database-connection-pooling.md) | Learning | Must Know | Easy |
| Database Fundamentals | [database-fundamentals.md](07-databases/database-fundamentals.md) | Learning | Must Know | Easy |
| Database Indexing | [database-indexing.md](07-databases/database-indexing.md) | Learning | Must Know | Medium |
| Database Keys | [database-keys.md](07-databases/database-keys.md) | Learning | Must Know | Easy |
| Locking (Optimistic / Pessimistic / Deadlock) | [database-locking.md](07-databases/database-locking.md) | Learning | Advanced | Medium |
| Isolation Levels and Anomalies | [isolation-levels.md](07-databases/isolation-levels.md) | Learning | Important | Medium |
| Normalization vs Denormalization | [normalization-vs-denormalization.md](07-databases/normalization-vs-denormalization.md) | Learning | Must Know | Medium |
| Query Optimization / Planner | [query-optimization.md](07-databases/query-optimization.md) | Learning | Important | Medium |
| Schema Migration / Evolution | [schema-migration.md](07-databases/schema-migration.md) | Learning | Important | Medium |
| Soft Delete / Audit Tables | [soft-delete-audit-tables.md](07-databases/soft-delete-audit-tables.md) | Learning | Important | Easy |
| SQL vs NoSQL | [sql-vs-nosql.md](07-databases/sql-vs-nosql.md) | Learning | Must Know | Easy |
| Transactions and ACID | [transactions-and-acid.md](07-databases/transactions-and-acid.md) | Learning | Must Know | Medium |
---
## 08 — Replication and Consistency [`Must Know · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| CAP Theorem | [cap-theorem.md](08-replication-and-consistency/cap-theorem.md) | Learning | Must Know | Hard |
| Logical / Lamport / Vector Clocks | [clocks-and-ordering.md](08-replication-and-consistency/clocks-and-ordering.md) | Learning | Advanced | Hard |
| Conflict Resolution (LWW / Version Vectors) | [conflict-resolution.md](08-replication-and-consistency/conflict-resolution.md) | Learning | Advanced | Hard |
| Consistency Models (Read-After-Write / Monotonic) | [consistency-models.md](08-replication-and-consistency/consistency-models.md) | Learning | Must Know | Medium |
| Database Replication | [database-replication.md](08-replication-and-consistency/database-replication.md) | Learning | Must Know | Medium |
| Failover | [failover.md](08-replication-and-consistency/failover.md) | Learning | Must Know | Medium |
| PACELC | [pacelc.md](08-replication-and-consistency/pacelc.md) | Learning | Advanced | Medium |
| Quorum / Majority Consensus | [quorum.md](08-replication-and-consistency/quorum.md) | Learning | Advanced | Hard |
| Replication Lag | [replication-lag.md](08-replication-and-consistency/replication-lag.md) | Learning | Must Know | Medium |
| Split Brain | [split-brain.md](08-replication-and-consistency/split-brain.md) | Learning | Advanced | Hard |
| Strong vs Eventual Consistency | [strong-vs-eventual-consistency.md](08-replication-and-consistency/strong-vs-eventual-consistency.md) | Learning | Must Know | Medium |
---
## 09 — Sharding and Partitioning [`Must Know · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Consistent Hashing | [consistent-hashing.md](09-sharding-and-partitioning/consistent-hashing.md) | Learning | Must Know | Medium |
| Cross-Shard Queries and Transactions | [cross-shard-queries.md](09-sharding-and-partitioning/cross-shard-queries.md) | Learning | Advanced | Hard |
| Data Migration (Dual Reads/Writes, CDC) | [data-migration.md](09-sharding-and-partitioning/data-migration.md) | Learning | Advanced | Hard |
| Partitioning vs Sharding | [partitioning-vs-sharding.md](09-sharding-and-partitioning/partitioning-vs-sharding.md) | Learning | Must Know | Easy |
| Rendezvous Hashing | [rendezvous-hashing.md](09-sharding-and-partitioning/rendezvous-hashing.md) | Learning | Advanced | Medium |
| Shard Key | [shard-key.md](09-sharding-and-partitioning/shard-key.md) | Learning | Must Know | Medium |
| Shard Rebalancing and Hot Shard | [shard-rebalancing.md](09-sharding-and-partitioning/shard-rebalancing.md) | Learning | Must Know | Medium |
| Shard Routing and Metadata | [shard-routing.md](09-sharding-and-partitioning/shard-routing.md) | Learning | Must Know | Medium |
| Sharding Strategies | [sharding-strategies.md](09-sharding-and-partitioning/sharding-strategies.md) | Learning | Must Know | Medium |
| Sharding | [sharding.md](09-sharding-and-partitioning/sharding.md) | Learning | Must Know | Medium |
| Virtual Nodes | [virtual-nodes.md](09-sharding-and-partitioning/virtual-nodes.md) | Learning | Advanced | Medium |
---
## 10 — Messaging and Event-Driven [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Consumer Lag | [consumer-lag.md](10-messaging-and-event-driven/consumer-lag.md) | Learning | Must Know | Medium |
| Ack / Visibility Timeout / Retry / DLQ | [delivery-and-retry.md](10-messaging-and-event-driven/delivery-and-retry.md) | Learning | Must Know | Medium |
| Delivery Semantics | [delivery-semantics.md](10-messaging-and-event-driven/delivery-semantics.md) | Learning | Must Know | Medium |
| Event-Driven Architecture | [event-driven-architecture.md](10-messaging-and-event-driven/event-driven-architecture.md) | Learning | Must Know | Medium |
| Event Sourcing and CQRS | [event-sourcing-cqrs.md](10-messaging-and-event-driven/event-sourcing-cqrs.md) | Learning | Advanced | Hard |
| Event Types: Notification / Carried State / Command vs Event | [event-types.md](10-messaging-and-event-driven/event-types.md) | Learning | Important | Medium |
| Dedup / Idempotent Consumer | [idempotent-consumer.md](10-messaging-and-event-driven/idempotent-consumer.md) | Learning | Must Know | Medium |
| Message Queue | [message-queue.md](10-messaging-and-event-driven/message-queue.md) | Learning | Must Know | Easy |
| Outbox Pattern | [outbox-pattern.md](10-messaging-and-event-driven/outbox-pattern.md) | Learning | Must Know | Medium |
| Publish/Subscribe | [publish-subscribe.md](10-messaging-and-event-driven/publish-subscribe.md) | Learning | Must Know | Easy |
---
## 11 — Kafka [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Kafka Architecture | [kafka-architecture.md](11-kafka/kafka-architecture.md) | Learning | Must Know | Medium |
| Kafka Cluster | [kafka-cluster.md](11-kafka/kafka-cluster.md) | Learning | Must Know | Medium |
| Kafka Delivery Guarantees | [kafka-delivery-guarantees.md](11-kafka/kafka-delivery-guarantees.md) | Learning | Advanced | Hard |
| Kafka Ecosystem (Connect / Streams / MirrorMaker) | [kafka-ecosystem.md](11-kafka/kafka-ecosystem.md) | Learning | Advanced | Medium |
| Kafka Operations (Capacity / Hot Partitions / Failure) | [kafka-operations.md](11-kafka/kafka-operations.md) | Learning | Advanced | Medium |
| Kafka Ordering | [kafka-ordering.md](11-kafka/kafka-ordering.md) | Learning | Must Know | Medium |
| Kafka Producers and Consumers | [kafka-producers-consumers.md](11-kafka/kafka-producers-consumers.md) | Learning | Must Know | Medium |
| Kafka Rebalancing | [kafka-rebalancing.md](11-kafka/kafka-rebalancing.md) | Learning | Must Know | Medium |
| Kafka Replication | [kafka-replication.md](11-kafka/kafka-replication.md) | Learning | Must Know | Medium |
| Kafka Retention | [kafka-retention.md](11-kafka/kafka-retention.md) | Learning | Important | Medium |
| Schema Registry and Serialization | [kafka-schema-registry.md](11-kafka/kafka-schema-registry.md) | Learning | Advanced | Hard |
---
## 12 — Reliability and Resilience [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Bulkhead | [bulkhead.md](12-reliability-and-resilience/bulkhead.md) | Learning | Important | Medium |
| Chaos Engineering | [chaos-engineering.md](12-reliability-and-resilience/chaos-engineering.md) | Learning | Advanced | Medium |
| Circuit Breaker | [circuit-breaker.md](12-reliability-and-resilience/circuit-breaker.md) | Learning | Must Know | Medium |
| Disaster Recovery | [disaster-recovery.md](12-reliability-and-resilience/disaster-recovery.md) | Learning | Must Know | Hard |
| Heartbeat and Health Checks | [heartbeat-health-checks.md](12-reliability-and-resilience/heartbeat-health-checks.md) | Learning | Must Know | Easy |
| Hedged Requests | [hedged-requests.md](12-reliability-and-resilience/hedged-requests.md) | Learning | Advanced | Medium |
| Idempotent Retry | [idempotent-retry.md](12-reliability-and-resilience/idempotent-retry.md) | Learning | Must Know | Medium |
| Load Shedding / Fail-Fast / Fallback | [load-shedding.md](12-reliability-and-resilience/load-shedding.md) | Learning | Important | Medium |
| Rate Limiter | [rate-limiter.md](12-reliability-and-resilience/rate-limiter.md) | Learning | Must Know | Medium |
| Retry and Timeout | [retry-and-timeout.md](12-reliability-and-resilience/retry-and-timeout.md) | Learning | Must Know | Medium |
| RPO and RTO | [rpo-rto.md](12-reliability-and-resilience/rpo-rto.md) | Learning | Must Know | Medium |
| Standby Models | [standby-models.md](12-reliability-and-resilience/standby-models.md) | Learning | Must Know | Medium |
---
## 13 — Deployment and Infrastructure [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Autoscaling | [autoscaling.md](13-deployment-and-infrastructure/autoscaling.md) | Learning | Must Know | Medium |
| CI/CD | [ci-cd.md](13-deployment-and-infrastructure/ci-cd.md) | Learning | Important | Easy |
| Cloud Infrastructure (Regions / AZs / VPC) | [cloud-infrastructure.md](13-deployment-and-infrastructure/cloud-infrastructure.md) | Learning | Must Know | Medium |
| Containers and VMs | [containers-and-vms.md](13-deployment-and-infrastructure/containers-and-vms.md) | Learning | Must Know | Easy |
| Deployment Strategies | [deployment-strategies.md](13-deployment-and-infrastructure/deployment-strategies.md) | Learning | Must Know | Medium |
| Feature Flags | [feature-flags.md](13-deployment-and-infrastructure/feature-flags.md) | Learning | Important | Easy |
| Infrastructure as Code | [infrastructure-as-code.md](13-deployment-and-infrastructure/infrastructure-as-code.md) | Learning | Important | Easy |
| Kubernetes Services and Ingress | [kubernetes-services.md](13-deployment-and-infrastructure/kubernetes-services.md) | Learning | Important | Medium |
| Kubernetes | [kubernetes.md](13-deployment-and-infrastructure/kubernetes.md) | Learning | Must Know | Hard |
| Serverless | [serverless.md](13-deployment-and-infrastructure/serverless.md) | Learning | Important | Medium |
| Service Discovery | [service-discovery.md](13-deployment-and-infrastructure/service-discovery.md) | Learning | Must Know | Medium |
| Service Mesh | [service-mesh.md](13-deployment-and-infrastructure/service-mesh.md) | Learning | Advanced | Hard |
---
## 14 — Observability [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Alerting and Alert Fatigue | [alerting.md](14-observability/alerting.md) | Learning | Important | Medium |
| Distributed Tracing | [distributed-tracing.md](14-observability/distributed-tracing.md) | Learning | Must Know | Medium |
| Golden Signals | [golden-signals.md](14-observability/golden-signals.md) | Learning | Must Know | Easy |
| Incident Management / On-Call / Postmortem | [incident-management.md](14-observability/incident-management.md) | Learning | Important | Medium |
| Structured / Centralized Logging | [logging.md](14-observability/logging.md) | Learning | Important | Easy |
| Observability | [observability.md](14-observability/observability.md) | Learning | Must Know | Medium |
| SLI / SLO / SLA | [sli-slo-sla.md](14-observability/sli-slo-sla.md) | Learning | Must Know | Medium |
| Synthetic Monitoring and Profiling | [synthetic-monitoring.md](14-observability/synthetic-monitoring.md) | Learning | Advanced | Medium |
---
## 15 — Security [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Access Control | [access-control.md](15-security/access-control.md) | Learning | Important | Medium |
| API Keys and Signed Session Tokens | [api-keys-sessions.md](15-security/api-keys-sessions.md) | Learning | Important | Medium |
| Authentication vs Authorization | [authentication-vs-authorization.md](15-security/authentication-vs-authorization.md) | Learning | Must Know | Medium |
| Data Masking and Privacy | [data-masking.md](15-security/data-masking.md) | Learning | Important | Easy |
| Encryption and Keys | [encryption-and-keys.md](15-security/encryption-and-keys.md) | Learning | Must Know | Hard |
| OAuth 2.0 / OIDC / JWT | [oauth-oidc-jwt.md](15-security/oauth-oidc-jwt.md) | Learning | Must Know | Hard |
| Password Hashing | [password-hashing.md](15-security/password-hashing.md) | Learning | Important | Medium |
| Tenant Isolation / Audit Trails / Zero Trust | [tenant-isolation.md](15-security/tenant-isolation.md) | Learning | Advanced | Hard |
| WAF and DDoS Protection | [waf-ddos.md](15-security/waf-ddos.md) | Learning | Must Know | Medium |
| Web Vulnerabilities | [web-vulnerabilities.md](15-security/web-vulnerabilities.md) | Learning | Must Know | Medium |
---
## 16 — Storage [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Blob Storage | [blob-storage.md](16-storage/blob-storage.md) | Learning | Important | Easy |
| Chunking and Resumable Uploads | [chunking-and-uploads.md](16-storage/chunking-and-uploads.md) | Learning | Must Know | Medium |
| Compression and Serialization | [compression.md](16-storage/compression.md) | Learning | Important | Easy |
| Content-Addressable Storage | [content-addressable-storage.md](16-storage/content-addressable-storage.md) | Learning | Advanced | Hard |
| Erasure Coding | [erasure-coding.md](16-storage/erasure-coding.md) | Learning | Advanced | Hard |
| File / Block / Object Storage | [file-block-object-storage.md](16-storage/file-block-object-storage.md) | Learning | Must Know | Easy |
| Immutable Storage and Versioning | [immutable-storage.md](16-storage/immutable-storage.md) | Learning | Important | Medium |
| Media Processing Pipeline | [media-processing.md](16-storage/media-processing.md) | Learning | Important | Medium |
| Storage Tiering and Lifecycle | [storage-tiering.md](16-storage/storage-tiering.md) | Learning | Must Know | Medium |
---
## 17 — Search and Data Processing [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Autocomplete | [autocomplete.md](17-search-and-data-processing/autocomplete.md) | Learning | Important | Medium |
| Batch vs Stream Processing | [batch-vs-stream-processing.md](17-search-and-data-processing/batch-vs-stream-processing.md) | Learning | Must Know | Medium |
| Data Warehouse and Data Lake | [data-warehouse-lake.md](17-search-and-data-processing/data-warehouse-lake.md) | Learning | Important | Hard |
| Elasticsearch | [elasticsearch.md](17-search-and-data-processing/elasticsearch.md) | Learning | Must Know | Hard |
| MapReduce, Lambda, and Kappa | [mapreduce-lambda-kappa.md](17-search-and-data-processing/mapreduce-lambda-kappa.md) | Learning | Advanced | Hard |
| OLTP vs OLAP | [oltp-vs-olap.md](17-search-and-data-processing/oltp-vs-olap.md) | Learning | Important | Medium |
| Probabilistic Data Structures | [probabilistic-data-structures.md](17-search-and-data-processing/probabilistic-data-structures.md) | Learning | Advanced | Hard |
| Search Engine | [search-engine.md](17-search-and-data-processing/search-engine.md) | Learning | Must Know | Medium |
| Search Ranking | [search-ranking.md](17-search-and-data-processing/search-ranking.md) | Learning | Important | Medium |
---
## 18 — Distributed Systems [`Must Know · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Byzantine Fault Tolerance | [bft.md](18-distributed-systems/bft.md) | Learning | Advanced | Hard |
| Consensus | [consensus.md](18-distributed-systems/consensus.md) | Learning | Advanced | Hard |
| CRDTs | [crdt.md](18-distributed-systems/crdt.md) | Learning | Advanced | Hard |
| Distributed ID Generation | [distributed-id-generation.md](18-distributed-systems/distributed-id-generation.md) | Learning | Must Know | Medium |
| Distributed Locks | [distributed-locks.md](18-distributed-systems/distributed-locks.md) | Learning | Must Know | Medium |
| Distributed Rate Limiter | [distributed-rate-limiter.md](18-distributed-systems/distributed-rate-limiter.md) | Learning | Advanced | Hard |
| Distributed Scheduling | [distributed-scheduling.md](18-distributed-systems/distributed-scheduling.md) | Learning | Advanced | Medium |
| Distributed Transactions (2PC / Saga) | [distributed-transactions.md](18-distributed-systems/distributed-transactions.md) | Learning | Must Know | Hard |
| Exactly-Once Effect | [exactly-once-effect.md](18-distributed-systems/exactly-once-effect.md) | Learning | Must Know | Hard |
| Gossip Protocol | [gossip-protocol.md](18-distributed-systems/gossip-protocol.md) | Learning | Advanced | Hard |
| Raft and Paxos | [raft-and-paxos.md](18-distributed-systems/raft-and-paxos.md) | Learning | Advanced | Hard |
---
## 19 — Architecture Patterns [`Must Know · Important · Advanced`]

| Concept                           | File                                                                                        | Status   | Priority  | Difficulty |
| --------------------------------- | ------------------------------------------------------------------------------------------- | -------- | --------- | ---------- |
| Data Access Patterns              | [data-patterns.md](19-architecture-patterns/data-patterns.md)                               | Learning | Must Know | Medium     |
| Fan-Out / Fan-In / Scatter-Gather | [fanout-and-aggregation.md](19-architecture-patterns/fanout-and-aggregation.md)             | Learning | Important | Medium     |
| Hexagonal and Clean Architecture  | [hexagonal-clean-architecture.md](19-architecture-patterns/hexagonal-clean-architecture.md) | Learning | Advanced  | Hard       |
| Layered / N-Tier Architecture     | [layered-architecture.md](19-architecture-patterns/layered-architecture.md)                 | Learning | Must Know | Easy       |
| Microservices                     | [microservices.md](19-architecture-patterns/microservices.md)                               | Learning | Must Know | Medium     |
| Modular Monolith                  | [modular-monolith.md](19-architecture-patterns/modular-monolith.md)                         | Learning | Important | Medium     |
| Monolith                          | [monolith.md](19-architecture-patterns/monolith.md)                                         | Learning | Important | Easy       |
| Publisher-Subscriber Pattern      | [pub-sub-pattern.md](19-architecture-patterns/pub-sub-pattern.md)                           | Learning | Must Know | Easy       |
| Resilience Patterns (Catalog)     | [resilience-patterns.md](19-architecture-patterns/resilience-patterns.md)                   | Learning | Must Know | Medium     |
| Saga and Strangler Fig            | [saga-and-strangler.md](19-architecture-patterns/saga-and-strangler.md)                     | Learning | Must Know | Medium     |
| Multi-Tenancy and Cell-Based Architecture | [tenancy-and-cells.md](19-architecture-patterns/tenancy-and-cells.md) | Learning | Advanced | Hard |
---
## 20 — Multi-Region Systems [`Must Know · Important · Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Cross-Region Replication | [cross-region-replication.md](20-multi-region-systems/cross-region-replication.md) | Learning | Must Know | Hard |
| Data Residency and Sovereignty | [data-residency.md](20-multi-region-systems/data-residency.md) | Learning | Important | Medium |
| Edge Computing | [edge-computing.md](20-multi-region-systems/edge-computing.md) | Learning | Important | Medium |
| Geo-DNS and Anycast | [geo-dns-anycast.md](20-multi-region-systems/geo-dns-anycast.md) | Learning | Must Know | Medium |
| Global Consistency | [global-consistency.md](20-multi-region-systems/global-consistency.md) | Learning | Advanced | Hard |
| Global Coordination | [global-coordination.md](20-multi-region-systems/global-coordination.md) | Learning | Advanced | Hard |
| Locality-Based Routing | [locality-based-routing.md](20-multi-region-systems/locality-based-routing.md) | Learning | Advanced | Medium |
| Active-Active vs Active-Passive Regions | [multi-region-models.md](20-multi-region-systems/multi-region-models.md) | Learning | Must Know | Medium |
| Regional Failover | [regional-failover.md](20-multi-region-systems/regional-failover.md) | Learning | Must Know | Medium |
---
## 21 — Advanced Concepts [`Advanced`]

| Concept | File | Status | Priority | Difficulty |
|---------|------|--------|----------|------------|
| Adversarial Reliability (Chaos at Scale) | [adversarial-reliability.md](21-advanced-concepts/adversarial-reliability.md) | Learning | Advanced | Hard |
| Cell-Based Architecture | [cell-based-architecture.md](21-advanced-concepts/cell-based-architecture.md) | Learning | Advanced | Hard |
| Multi-Region Consensus | [multi-region-consensus.md](21-advanced-concepts/multi-region-consensus.md) | Learning | Advanced | Hard |
| Deterministic Systems and Replayability | [replayability.md](21-advanced-concepts/replayability.md) | Learning | Advanced | Hard |
| Serverless at Scale | [serverless-at-scale.md](21-advanced-concepts/serverless-at-scale.md) | Learning | Advanced | Hard |
| Predictable Tail Latency | [tail-latency.md](21-advanced-concepts/tail-latency.md) | Learning | Advanced | Hard |
| Time Series at Scale | [time-series-at-scale.md](21-advanced-concepts/time-series-at-scale.md) | Learning | Advanced | Hard |
---

## Planned (future work) — concept lines covered by an existing file

The following syllabus rows were never authored as separate files; the owning concepts already exist and are linked from the sections above.

- `monolith`, `modular-monolith`, `microservices` → `19-architecture-patterns/monolith.md`, `modular-monolith.md`, `microservices.md`
- `auto-scaling` (05) → `13-deployment-and-infrastructure/autoscaling.md`
- `producer-consumer`, `topics-partitions-offsets` (10) → `10-messaging-and-event-driven/message-queue.md`, `publish-subscribe.md` and `11-kafka/kafka-cluster.md`, `kafka-producers-consumers.md`

## Status Rollup

- **Interview ready:** 0
- **Revised:** 0
- **Learning (authored, needs review):** 247
- **By priority:** Must Know 121 · Important 69 · Advanced 57
- **By difficulty:** Easy 65 · Medium 140 · Hard 42

## Revision Hub

Rapid-recall and decision-framework files live in `../Revision/` — one-liner recap, then decision trees per layer (database / caching / messaging / reliability), then the interview checklist.

## Generated Files

- [progress.md](progress.md) — learning-status tracker (per-concept status, priority, difficulty)
- `../Revision/` — `01-rapid-revision`, `02-database-decisions`, `03-caching-decisions`, `04-messaging-decisions`, `05-reliability-decisions`, `06-hld-interview-checklist`
- [00-obsidian-readability.md](00-obsidian-readability.md) — template and format rationale
