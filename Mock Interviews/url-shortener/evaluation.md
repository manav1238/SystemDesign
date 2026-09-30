---
title: "Design a URL Shortener — Interview Evaluation"
status: active
date: 2026-09-29
tags: [hld, mock, url-shortener, evaluation]
---

# Design a URL Shortener — Interview Evaluation

## Scorecard

| Phase | Score | Why |
|---|---|---|
| Phase 1 — Requirements Clarification | Excellent | Asked about long-URL length, alias uniqueness scope, destination mutability, and click freshness before estimating; each one visibly changed the data model or the consistency model. |
| Phase 2 — Estimate | Good | Arithmetic was correct and shown (1M writes/day to about 35 QPS, 1B reads/day to about 120K peak QPS, 870 bytes per row to about 90 GB primary), but peak QPS was assumed at 10x average rather than derived from a traffic shape, and bandwidth was treated as negligible. |
| Phase 3 — High-level design | Excellent | Clear diagram, explicit control-plane versus data-plane split, and the read path enumerated step by step with the Bloom filter placed correctly between cache and database. |
| Phase 4 — Deep dive | Excellent | Base62 collision math shown properly (about 1,400 expected collisions at 7 chars with 100M keys, therefore 9 chars for safety), key pool justified over random retry, and the 301/302 argument made from click-counting, revocability, and method preservation. |
| Phase 5 — Trade-offs and follow-ups | Good | All the prompted scenarios were handled with real numbers, but the sweeper was flagged as a weakness only late, and the sharding decision was justified by throughput rather than by a stated cost comparison against staying single-shard. |

## What Made This a Strong Answer

- **Refused to accept the framing of a single system.** Splitting the control plane from the data plane early, then refusing to add analytics to the redirect path, is the decision that makes everything else fit.
- **Ran the collision math instead of hand-waving it.** Computing 62^7 and then showing that the birthday bound makes random generation wrong at 100M keys, and therefore proposing a pre-generated key pool, is the single most differentiating moment in the transcript.
- **Named the health of the system as one number.** Reducing the whole design to "the cache hit rate holds or it does not" is the kind of compression that signals real operational experience.
- **Understood that a Bloom filter is dangerous, not just useful.** Identifying that an incompletely loaded filter converts a slow answer into a wrong answer, and requiring generation-based atomic swap with a bypass when not ready, shows awareness of how probabilistic structures fail in production.
- **Defended the 301 versus 302 decision on three independent grounds** rather than defaulting to a convention, and tied the opt-in flag to the analytics and revocability consequences.
- **Quantified failures instead of narrating them.** "1 in 64 of codes return 503" and "17,000 requests per second on one B-tree page" are the right units for a capacity conversation.

## Memory Hooks

- Split control plane from data plane before estimating anything.
- Base62, 62^7 is 3.5 trillion, and the birthday bound kills you at 100M keys; pre-generate a key pool instead of retrying.
- Database is the source of truth, cache is disposable and derived, and a Bloom filter only ever converts a miss into a slower path, never into a wrong one.
- 302 by default, 301 only as an explicit opt-in that trades click counts and revocability for edge cost.
- Expiry guarantee lives on the read path, cache TTL is capped at expires_at, and the sweeper is cleanup, not correctness.
- Shard by short code so losing one primary is 1-in-64 of traffic, not an outage.
- Whole-system health equals cache hit rate; a viral key is handled by L1 plus pre-warming, not by hoping.

## Weak-Spot Pointers

- [[consistent-hashing|Consistent Hashing]] — you dodged the question with a fixed modulo shard count. Know when modulo stops working, what the rebalancing cost is when you go from 64 to 128 shards, and how virtual nodes bound it to one in N keys.
- [[probabilistic-data-structures|Probabilistic Data Structures]] — drill the counting Bloom filter, the sizing formula for 10 bits per key, and specifically the false-negative versus false-positive asymmetry your design depends on.
- [[http-caching|HTTP Caching]] — the 302 answer was right but the surrounding edge caching reasoning was thin. Be able to argue cache key design, TTL, and why `no-store` is not always the right header.
- [[normalization-vs-denormalization|Normalization vs Denormalization]] — the denormalised click count needs a stated staleness contract and a story for who repairs it when the aggregator falls behind.
- [[replication-lag|Replication Lag]] — you solved read-after-write with write-through and a hot set, but you did not quantify the actual lag distribution you are defending against.
- [[idempotency|Idempotency]] — the `Idempotency-Key` was correct but you never stated retention, scope, or what happens on a key collision.

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the five-phase skeleton this transcript was scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before an interview
- [[caching|Caching]] — the component carrying the entire read path
- [[sharding|Sharding]] — the decision hardest to reverse, and the one you justified too quickly
