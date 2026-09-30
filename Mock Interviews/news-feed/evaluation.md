---
title: News Feed System Design - Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, news-feed, evaluation]
---

# News Feed System Design - Evaluation

## Scoring

| Phase | Score | Why |
|---|---|---|
| Phase 1: Requirements | Excellent | Separated ranked from chronological, pinned down the follower distribution at the tail, and explicitly wrote down the freshness, latency, and cost constraints instead of assuming them. |
| Phase 2: Estimate | Excellent | Did the arithmetic out loud with real numbers and landed the decisive one: a single post can require 9e7 fanout writes, which is what forces the hybrid model rather than a preference for it. |
| Phase 3: High-level design | Excellent | Four planes with a clean ASCII diagram, per-path walkthrough, and a different shard key justified for each store rather than "shard by user_id" everywhere. |
| Phase 4: Deep dive | Excellent | Fanout models drawn and defended, the celebrity threshold explained as a cost decision rather than a rule of thumb, ranking shown as precomputed-plus-cheap-correction, and the cursor design explained with a reason. |
| Phase 5: Trade-offs and follow-ups | Good | Seven trade-offs named with their cost, and every push-back handled. The one gap is that the ranking evaluation harness was described at a high level rather than concretely. |

**Overall: strong hire signal.** The distinguishing move was computing the tail before designing, so the architecture was forced by arithmetic instead of chosen by taste.

---

## What Made This a Strong Answer

- The 9e7-writes-per-post calculation came before the architecture. That single number pre-decided the pull/push/hybrid question and made every later decision look forced rather than arbitrary.
- Shard keys were chosen per store for the access pattern, and the cross-shard cost of the follow path was acknowledged out loud instead of being hidden by "everything is sharded by user_id".
- The feed was treated as a disposable read model with Kafka as the durable source of truth, which turned "what if the feed store dies" into "replay the log" instead of a data-loss conversation.
- Consistency was treated as a per-read property: the feed tolerates replica lag, the follow graph goes to the primary, and the user's own post gets an optimistic local append. That is far more mature than "eventual everywhere" or "strong everywhere".
- The celebrity problem was answered with a threshold *and* an explanation of why raising the threshold does not fix the worst case, which is the part most candidates miss.
- The SQL design used an index strategy matched to the query, including a flattened thread read and a depth cap, and the hydrate step was collapsed into one batched multi-get rather than twenty point reads.

---

## Memory Hooks

- 3e10 reads a day, 1.5e8 writes a day, and the write side is cheap until you multiply it by followers.
- One celebrity post equals 9e7 writes. That is the whole design in a single number.
- Push for the many, pull for the few. Threshold around 10k followers, and raising it does not save you from the top 1%.
- The feed is a cache and a read model, never a source of truth. Kafka is the truth.
- Shard the feed by user_id so the read is one key; shard follows by follower so the graph read is one key too; never use the same key for both.
- Score once per post globally, personalize cheaply per reader. Two-stage ranking, same as search.
- Precompute before the event, pre-warm before the spike, and never let 50,000 identical misses become 50,000 identical assemblies.
- Cursors bind to the user id. An unscoped cursor is a data leak, not just a pagination bug.

---

## Weak-Spot Pointers

- **Ranking evaluation was hand-wavy.** "I'd use NDCG and shadow a GBDT" is directionally right but not an answer. Have a concrete offline metric, a concrete online metric, and a concrete guardrail metric ready, and be able to name the negative-signal metric that catches a greedy-ranking regression. Drill [[search-ranking]].
- **Multi-region was deferred too quickly.** "Geo DNS and per-region caches" is one sentence. Be ready to say which reads are local, how a follow in region A reaches region B, and what the consistency cost is. Drill [[geo-dns-anycast]] and [[global-consistency]].
- **Cost estimation was asserted, not computed.** The transcript says fanout is a cost multiplier but never puts a dollar or CPU-second number on the 4.5e10 daily feed-item writes. Interviewers ask this. Drill [[cost-estimation]] and [[server-capacity]].
- **The social graph deserves its own deep dive.** At 3e12 edges it becomes a 60 TB problem with a hot "who follows me" scan. That was waved away in one sentence at 100x. Drill [[cross-shard-queries]] and [[sharding-strategies]].
- **No mention of the read-path time budget breakdown.** 200 ms p99 was asserted, but a strong answer spends it: 5 ms edge, 20 ms Redis, 30 ms hydration, 50 ms network, 95 ms slack. Drill [[latency-budget]] and [[tail-latency]].

---

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] - the phase-by-phase checklist to run through before you walk in
- [[01-rapid-revision|Rapid Revision]] - the full vault compressed into a revision pass
- [[fanout-and-aggregation|Fanout and Aggregation]] - the single most reused pattern in this problem
- [[shard-key|Shard Key]] - the decision that makes or breaks the read path
- [[kafka-architecture|Kafka Architecture]] - the event log that makes the read model replayable
- [[hotspot-handling|Hotspot Handling]] - celebrity and hot-counter techniques in one place
