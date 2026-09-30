---
title: "Design Uber — Evaluation and Scoring"
status: active
date: 2026-09-29
tags: [hld, mock, uber, evaluation]
---

# Design Uber — Evaluation and Scoring

Session reviewed against the five phases in [[problem|Problem Statement]] and [[06-hld-interview-checklist|HLD Interview Checklist]].

## Scoring

| Phase | Score | Why |
|-------|-------|-----|
| 1. Requirements Clarification | 10/10 | Opened on the device edge and won the single most valuable answer of the session: adaptive location reporting is a cost lever, not a constraint. Split freshness into three tiers, 400ms rider map, 2s matching index, minutes for history, before estimating anything. Then asked whether a driver offer is exclusive, which reframed matching from a search problem into a compare-and-set problem. |
| 2. Scale Estimation | 10/10 | Split 5M drivers into on-trip and idle, weighted reporting rates, and got 500K pings/s inbound with a second-method check at 43.2B pings/day and 3.45 TB/day. Then found the number that sizes the system: 1.5M outbound fan-out messages/s, 3x the ingest, derived from 1.25M concurrent ride watchers at 1Hz. The "the database is the easy part" framing is the correct read of the workload. |
| 3. API and Data Design | 9/10 | Clean BFF split, a real WebSocket contract with channels and rate_hz, delta-encoded seq-stamped frames, and idempotency keys placed on the trip event and notification paths. The single-statement CAS with `rows_affected = 0` and the partial unique index on live assignments are the two designs that make the double-assignment bug structurally impossible. Slightly thin on the surge service's own contract. |
| 4. High-Level Architecture | 9/10 | Five independent scale units drawn explicitly, and the diagram shows the boundary that matters: nothing on the location path touches Postgres. The 10-step ingest path with per-step latency budget, and the 9-step match path with CAS-before-notify ordering, are the two flows worth memorizing. Lost points for treating the notification path as fire-and-forget without a delivery-receipt story. |
| 5. Trade-offs and Failure Scenarios | 9/10 | GPS-rate trade-off argued with real arithmetic and answered with adaptive rate plus a last-200-meters rule and client dead reckoning, including the honest limitation that extrapolated positions must never feed the index. All six pushbacks answered: 2x traffic (shed idle pings, degrade pre-request map, shed by queue age), primary death (narrow blast radius because matching never reads the trip DB), hot cell (5 mitigations ordered by shippability), replica lag (never verify on a replica; unique index as the backstop), stampede (made impossible by dataset size, then the real mass-expiry case, surge versioning). |
| **Overall** | **48/50** | Hire, strong hire. The offer-exclusivity question and the fan-out estimate were the two moments that separated this from a competent generic ride-hailing answer. |

## What Made This a Strong Answer

- Asked whether a driver can hold two pending offers before designing the matcher, and correctly concluded that the answer converts the whole problem from a search into a compare-and-set. Almost nobody asks this and it changes the architecture.
- Found the fan-out number. 1.5M outbound messages per second against 500K inbound is the estimate that determines the realtime tier's shape, and it is invisible if you only count what arrives at your servers.
- Argued that the transactional core is the easy part and said so explicitly, which is counter-intuitive for a problem whose framing is "the database of record." 7,000 writes/s across 256 shards is 27 writes per shard, and naming that stops the interview going in the wrong direction.
- Made mutual exclusion a database invariant rather than an application convention: one compare-and-set, no read-then-write window, plus a partial unique index that rejects a bad assignment even if the application logic is wrong. That is the difference between a bug and a data-integrity incident.
- Answered the GPS-rate pushback with a cost calculation and a mechanism instead of a preference, including the 10x-ingest-for-2x-accuracy point and the observation that a 4-second fix at highway speed already moves 44 meters, which is worse than the urban-canyon measurement error you were trying to fix.
- Named the failure modes it would actually page for and admitted that none of them are the ones the problem statement suggests. Hot cell, stale driver state, and connection-count growth are the three real risks in this system.

## Memory Hooks

- Two freshness tiers, not one: 400ms rider map, 2s matching index, minutes for history. Only the first is a hard latency path, so only the first earns strong consistency.
- Location ingest split by driver state: 1.5M on-trip at 4s plus 3.5M idle at 30s equals 500K pings/s. Never divide by one interval.
- Fan-out is 1.5M msg/s outbound, three times ingest. Design the path that tells riders where drivers are, not the path that collects it.
- 7,000 transactional writes/s over 256 shards is 27 per shard. The database is the easy part here and saying it out loud is the move.
- Nothing on the location path touches Postgres: phone to broker is 55ms, broker to index is another 55ms, and one row write would spend the whole 400ms budget.
- Offer exclusivity turns matching into a CAS. One `UPDATE ... WHERE availability = 0`, check `rows_affected`, back a partial unique index on live assignments.
- Cache TTL must never exceed the lifetime of the thing cached. Fare quote TTL is bounded by the surge multiplier's validity window, and multiplier changes are rate-capped so mass invalidation becomes a trickle.
- Hot region is a contention problem, not a capacity problem. Fix: adaptive cell subdivision, then widen the ring instead of retrying the same drivers.
- Stampede defense in order of strength: make the dataset small enough that there is no miss path, then TTL jitter, then single-flight, then stale-while-revalidate.
- Adaptive reporting rate beats uniform 1Hz. Full rate only inside 200 meters of a pickup or during a dispute replay, and dead reckoning is a rendering concern that must never feed the index.

## Weak-Spot Pointers

- Surge computation was named as in-scope in the questions and then only sketched. The real content is the supply/demand estimator, the half-life on its inputs, the price-elasticity feedback loop where surge drives drivers away and suppresses demand, and why the multiplier is rate-capped. Expect "walk me through surge" as the follow-up. Drill [[bottleneck-identification|Bottleneck Identification]] and [[time-series-at-scale|Time Series at Scale]].
- The geospatial index was described but not defended against alternatives at depth. Be ready to argue hex versus quadtree versus geohash versus PostGIS, and to say what breaks when you leave dense cities. Drill [[database-indexing|Database Indexing]] and [[shard-rebalancing|Shard Rebalancing]].
- Multi-region was only touched via the home-region pinning argument, which was correct but brief. The follow-up will be about a driver crossing a region border mid-trip, and about data residency in India and the EU, which you flagged and never closed. Drill [[multi-region-models|Multi-Region Models]] and [[data-residency|Data Residency]].
- Sharding was justified for throughput but not for rebalancing. 256 trip shards and 4,096 driver shards were asserted; nobody asked how you add shards without downtime or what the double-write looks like. Drill [[shard-rebalancing|Shard Rebalancing]] and [[data-migration|Data Migration]].
- Notification delivery was treated as best-effort. The push path has a real failure story: FCM and APNs both silently drop, offer expiry is a server-side timer that must survive a gateway restart, and SMS fallback is a paid third-party dependency with its own rate limit. Drill [[delivery-and-retry|Delivery and Retry]] and [[bulkhead|Bulkhead]].

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the five-phase skeleton this session was scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before the interview
- [[problem|Problem Statement]] — the prompt, clarifying questions, and evaluation criteria
- [[interview|Interview Study Transcript]] — the full dialogue
- [[shard-routing|Shard Routing]] — location index sharding by geocell
- [[distributed-locks|Distributed Locks]] — the one-driver-one-rider invariant
