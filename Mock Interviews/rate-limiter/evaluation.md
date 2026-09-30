---
title: "Design a Rate Limiter — Interview Evaluation"
status: active
date: 2026-09-29
tags: [hld, mock, rate-limiter, evaluation]
---

# Design a Rate Limiter — Interview Evaluation

## Scorecard

| Phase | Score | Why |
|---|---|---|
| Phase 1 — Requirements Clarification | Excellent | Asked about library versus service, abuse versus cost control, endpoint weighting, multi-dimension keys, and regional versus global quota; each answer changed the architecture rather than just the config. |
| Phase 2 — Estimate | Excellent | Showed the sliding window log is infeasible (about 1.08B entries per hour, roughly 43 GB) and therefore the algorithm must be constant-memory, sized shard count from Lua throughput (300K RPS to 24 shards), and proved local-only limiting is wrong by a factor of instance count. |
| Phase 3 — High-level design | Excellent | Three-tier diagram with the leased-quota insight, correct placement after auth and before business logic, platform ceiling moved to the load balancer and connection pool, and the API contract expressed as a decision object rather than an HTTP call. |
| Phase 4 — Deep dive | Good | Token bucket defended against all four alternatives with the weighting requirement as the decisive argument, atomic script and hash tags explained well, but the lease sizing arithmetic was revised mid-answer and the multi-dimension AND was asserted rather than worked through. |
| Phase 5 — Trade-offs and follow-ups | Excellent | Rejected both fail-open and fail-closed as global policies, proposed degraded local quota as the real answer, and handled partition, hot key, bad config, and retry storm with bounded-over-admission reasoning. |

## What Made This a Strong Answer

- **Found the actual problem.** The stated task was "design a rate limiter"; the real task was "300,000 decisions per second without 300,000 network round trips per second", and the leased-quota model is the answer that reframes it correctly.
- **Chose the algorithm from a calculation, not a preference.** Showing the sliding window log costs 43 GB per hour, and therefore constant-memory is mandatory, is the difference between knowing an algorithm and knowing when to use it.
- **Refused a binary on the headline question.** Per-route fail policy, plus degraded local limits sized as global limit divided by instance count, is a real engineering answer that most candidates never reach.
- **Quantified every failure.** Bounded over-admission under partition as "number of partitioned instances times the local quota" turns an outage narrative into a number you can put in an SLO.
- **Explained why a distributed lock is the wrong primitive here** and stated the failure mode of the naive alternative, which is the question interviewers actually ask to separate readers from memorisers.
- **Named the design's own weak point.** Flagging the degraded-mode constant as a guess and the leased model as rude to bursty enterprise tenants is the mark of someone who has operated this.

## Memory Hooks

- Local-only limiting multiplies the limit by the instance count, so with 300 instances a 100 RPS limit becomes 30,000 RPS.
- Leased quota turns 300,000 RPS into roughly 1,000 to 4,000 backend operations per second; that is the entire scalability story.
- Sliding window log is O(limit) memory and dies at 1.08B entries per hour; constant memory is not a preference, it is forced.
- Token bucket wins because it is the only candidate that expresses endpoint cost as tokens and bounds burst by capacity.
- One round trip, one script, one clock, hash tags on every key a script touches; two round trips is a race at 300,000 RPS.
- Fail closed on auth and payments, fail open on reads, degraded local quota in between; never one global policy.
- If the limiter can be the outage, the 5 ms timeout and circuit breaker are part of the design, not an optimisation.

## Weak-Spot Pointers

- [[distributed-rate-limiter|Distributed Rate Limiter]] — the core of the problem. Drill the leased-quota variant, its worst-case over-admission, and the alternative designs you did not consider such as centralised token leasing per tenant.
- [[clocks-and-ordering|Clocks and Ordering]] — you asserted clamping on negative and positive elapsed. Know exactly what a forward NTP step does to a token bucket and what Redis TIME does versus a monotonic clock.
- [[redis|Redis]] — Cluster slotting, hash tags, EVAL versus Functions, the RATE-LIMIT cell module, and what happens to a stateful counter during a failover.
- [[overload-protection|Overload Protection]] — you moved the platform ceiling to the load balancer and connection pool but did not specify the mechanism; a semaphore plus shed thresholds at the L7 layer is the answer.
- [[cap-theorem|CAP Theorem]] — the lease model is a deliberate availability-versus-consistency trade. You did not say it out loud, and a partition interviewer will make you.
- [[rate-limiter|Rate Limiter]] — re-derive fixed window versus sliding counter versus token bucket boundary behaviour from first principles rather than from memory.

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the five-phase skeleton this transcript was scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before an interview
- [[rate-limiter|Rate Limiter]] — the concept this interview drills
- [[distributed-rate-limiter|Distributed Rate Limiter]] — where the remaining weak spots live
