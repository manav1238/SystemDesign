---
title: "Design a Rate Limiter (study transcript)"
status: active
tags: [hld, mock, rate-limiter]
---

# Design a Distributed Rate Limiter — Problem Statement

## Problem Statement

Design a distributed rate limiting service that every API in a large internet platform must call before doing any work.

**Core product surface:**

- Clients are limited per user, per tenant or organization, per IP, and per API key, with different quotas for different subscription tiers.
- Limits are specified as a rate plus a burst allowance, for example "100 requests per second, bursts up to 300."
- Endpoints have different cost weights: a cheap read costs 1 token, an expensive search costs 10, a bulk export costs 100.
- When a client exceeds its quota, the request is rejected immediately with HTTP 429 and a `Retry-After` hint, before any business logic runs.
- The platform also needs platform-wide safety limits, such as a hard ceiling on total platform traffic, separate from any single user's quota.
- Limits must be enforceable across hundreds of horizontally scaled application instances, not just within one process.

**The interviewer says:**

> "Assume 200 million registered users, 20 million daily active users, and a peak aggregate of 300,000 requests per second flowing through the platform. The standard authenticated quota is 100 requests per second per user with a burst of 300; anonymous and unauthenticated traffic is limited per IP. Design the limiter. I want to know which algorithm you pick and why you reject the others, whether the state lives in memory or in a shared store, how you make it correct across hundreds of instances, what you do when the shared store dies, and how you size it. Also decide: do you fail open or fail closed?"

## Clarifying Questions You Should Ask

Ask these before you draw a single box. Each one signals that you are scoping the problem deliberately rather than guessing.

### Product scope

1. Is the limiter a network service that every call crosses, or a library or sidecar running next to the service? This is the single biggest latency and availability decision in the problem.
2. What is a request worth? Do all endpoints cost the same, or do we need weighted tokens where a search costs more than a read?
3. Which dimensions are limited: user, tenant, IP, API key, device, subscription tier, or a combination? Is the effective limit the minimum of several independent buckets?
4. Do quotas vary per endpoint, per tier, and per customer contract? Do enterprise customers get custom, higher limits?
5. Is the limiter used for abuse prevention, cost control, or protecting a downstream dependency? The answer decides fail-open versus fail-closed.
6. Do we notify clients of their quota, or just reject? Do we expose remaining-quota headers for client-side backoff?
7. Do we count retries as separate requests? A retry storm is exactly the situation where a limiter earns its keep.
8. Do we need distributed limits across regions, or is a quota scoped to one region? Cross-region global quotas are much harder.

### Scale and traffic

9. What is the aggregate peak requests per second through the limiter? Every one of those is a state read and possibly a decrement.
10. How many distinct keys are active in a given window? This is the difference between a few megabytes of state and an expensive one.
11. How many application instances will call the limiter? This determines whether a purely local limiter is even close to correct.
12. What is the distribution of traffic per key? Is it one heavy tenant generating 20 percent of traffic, or is it flat?
13. What is the required decision latency? I intend to argue for a p99 in the low single-digit milliseconds and a hard timeout of a few milliseconds.
14. What is the acceptable over-admission rate? Exact, or is a 1 percent overshoot acceptable in exchange for much higher throughput?

### Non-functional requirements

15. What is the availability target of the limiter itself? A limiter that takes down the platform when it fails has a design bug, not a capacity problem.
16. When the limiter's backend is unavailable, should the platform allow traffic (fail open) or reject it (fail closed)? Who decides, and can it be per-route?
17. How much over-admission is tolerable during a limiter outage? Zero for authentication endpoints is a very different promise from zero for static assets.
18. How quickly must a limit policy change take effect? Minutes, or seconds? This decides whether config lives in the database, in a feature flag, or is pushed to every instance.
19. Do we need exact accounting for billing or metering, or is this best-effort? Metering and throttling have opposite accuracy requirements.

### Constraints and preferences

20. Is a shared store such as [[redis|Redis]] available and acceptable on the request path?
21. Is eventual accuracy acceptable, or must the limit be strictly enforced across all instances at all times?
22. Can we use a local component such as an embedded library or sidecar, or is a central service mandated by the platform?
23. Do we need per-endpoint overrides and gradual rollout of new limits, such as via feature flags?
24. What metrics do we need for customers, such as usage dashboards and near-quota alerts?

## What You Are Evaluated On

The interview is scored on five phases, matching [[06-hld-interview-checklist|HLD Interview Checklist]].

### Phase 1 — Requirements Clarification

Did you ask scoping questions before estimating? Did you separate functional from non-functional requirements explicitly, and did you surface the fail-open versus fail-closed question as a product decision rather than an implementation detail? Did you ask about weighted endpoint cost and multi-dimensional keys, which most candidates forget? See [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]] and [[rate-limiter|Rate Limiter]].

### Phase 2 — Scale Estimation

Did you show arithmetic with real numbers: aggregate peak RPS, number of active keys, state size per key, total memory, and how many backend shards you need? Did you compare the memory cost of an exact sliding window log against a counter-based approach and show why the exact version is infeasible? Did you reason about what happens if the limit is enforced locally on N instances? See [[capacity-estimation|Capacity Estimation]], [[memory-estimation|Memory Estimation]], and [[server-capacity|Server Capacity]].

### Phase 3 — High-Level Architecture

Did you draw a clear diagram showing where the limiter sits relative to the load balancer, the application, and the shared store? Did you include a local in-process component, and justify it? Did you place it before business logic and after authentication, and did you show the [[circuit-breaker|Circuit Breaker]] and timeout around the backend call so the limiter can never become the outage? See [[rate-limiter|Rate Limiter]], [[overload-protection|Overload Protection]], and [[latency-budget|Latency Budget]].

### Phase 4 — Deep Dive

Did you compare fixed window, sliding window log, sliding window counter, token bucket, and leaky bucket with memory, precision, and burst behaviour, and then defend one choice? Did you go deep on the atomicity of the check-and-decrement, using a single [[redis|Redis]] `EVAL` script or equivalent, and explain why two round trips is a race? Did you cover monotonic versus wall clocks, key design with hash tags, approximate counting, leased local quota to amortize backend calls, and the local-versus-distributed accuracy trade-off? Did you cover multi-dimension keys, weighted tokens, and policy propagation? See [[distributed-rate-limiter|Distributed Rate Limiter]], [[distributed-locks|Distributed Locks]], and [[clocks-and-ordering|Clocks and Ordering]].

### Phase 5 — Trade-offs and Failure Scenarios

Did you give a nuanced fail-open versus fail-closed answer with a per-route policy rather than one global choice? Did you walk through concrete failures: backend shard death, a backend network partition, one hot key spraying traffic, clock skew from NTP, a bad limit config that throttles the whole platform, and a retry storm? Did you explain the cascading-failure risk of a synchronous limiter and how the timeout and bulkhead prevent it? See [[failover|Failover]], [[retry-and-timeout|Retry and Timeout]], [[bulkhead|Bulkhead]], and [[trade-off-analysis|Trade-Off Analysis]].

## Related Reading

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the skeleton this problem is scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before
- [[rate-limiter|Rate Limiter]] — the concept this interview drills
- [[distributed-rate-limiter|Distributed Rate Limiter]] — the multi-instance correctness problem
