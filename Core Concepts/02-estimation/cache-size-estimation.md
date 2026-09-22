---
title: Cache Size Estimation
category: Estimation
priority: must-know
status: learning
difficulty: medium
interview_ready: false
tags:
  - hld
  - estimation
  - caching
---

# Cache Size Estimation

## 1. One-Line Definition
Cache size estimation figures out how many distinct objects and bytes a cache must hold to serve the hot working set from memory — key space × value size × structure overhead × replication — so you can size Redis/Memcached nodes instead of guessing RAM.

## 2. Why Do We Need It?
A cache that is too small evicts the very keys you need (cache thrash → DB load), and a cache that is too big wastes the most expensive resource. Cache sizing also decides the *type* of cache: whether the working set fits one node (single Redis) or needs [[consistent-hashing|Consistent Hashing]] sharding, and whether it is even worth caching at all. "How big is the hot set?" is the first question after "cache or not?".

## 3. Simple Intuition
A library's popular-shelf after a renovation: you must choose how many shelves of books get promoted near the entrance. If you promote only 10% of the daily-requested titles, people still queue at the desk (misses). Cache sizing is deciding that shelf size — the number of titles multiplied by average book size, plus the fact that some shelf space goes to labels and gaps (structure overhead in Redis/Memcached).

## 4. What Happens Without It?
Two classic failures: (1) undersized — hit ratio collapses at the worst moment, every miss storms the DB, and you discover the cache is too small during the peak you built it for; (2) oversized — you buy a 512 GB Redis cluster for a 4 GB working set and still hit hot-key contention. Both feel like "caching didn't work" when the true fault was that the size was never calculated.

## 5. Core Idea
- **The working set, not the whole dataset, is what you size.** Cache the fraction of data that receives the traffic: `cache bytes = hot keys × bytes per value × (1 + structure overhead)`.
- **Key space:** how many distinct keys does the traffic touch in the TTL/eviction window? Feed examples: top-N posts, user-bucket feeds, product pages.
- **Value size is not payload size:** Redis strings have a per-key header, dict structures hold pointers, serialization adds framing — a 200 KB "value" may cost 260 KB resident.
- **Numbers to internalize:**
  - A "small" cached record (JSON, ~200-500 B) runs ~1 KB of cache RAM once Redis/dict overhead lands.
  - Per-key fixed overhead in Redis ≈ tens of bytes; in Memcached, slab overhead and the 1 MB value cap matter.
- **Replication multiplies:** caching in N replicas/fleet-local caches costs N copies of the same hot set — see [[replication-overhead|Replication Overhead]]. A shared cache stores the set once.
- **Eviction is a design input, not an accident:** with LRU, size = working-set estimate × a headroom factor (1.2-1.5x) so the hot keys survive natural churn.
- **TTL turnover adds little to *size*** but raises write load (misses to repopulate + expiry churn) — the two are siblings but separate: [[memory-estimation|Memory Estimation]] covers the churn/throughput side.

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| Working set | The data actually requested in a time window |
| Hot keys | Keys carrying most of the traffic |
| Key space / cardinality | Number of distinct keys hit |
| Value bytes | Payload size before structure overhead |
| Structure overhead | Headers, dict pointers, slab/waste memory |
| Effective size | Real RAM a cached value consumes |
| Hit ratio | % of requests served from cache |
| Thrash | Constant eviction of hot keys → misses |

## 7. Basic Architecture

```mermaid
flowchart LR
    R[Read QPS] --> H[Hot set analysis]
    H --> K[Distinct hot keys]
    K --> S[Key bytes plus value bytes]
    S --> O[Structure overhead]
    O --> E[Effective cache bytes]
    E --> N[Cache nodes]
    N --> R
```

## 8. Request or Data Flow
1. Enumerate the cached operations and their key shapes (`user:1:profile`, `feed:region:page`).
2. Count distinct keys in the traffic window (analytics, or sample keyspace) — this is cardinality.
3. Average value bytes × cardinality = payload bytes; add per-key + dict overhead (×1.2-2).
4. Add replication factor (1 for shared, N for local caches on N boxes) and 25-50% headroom.
5. Convert to node count by per-node RAM; sanity-check the hit ratio against the estimate before buying.

## 9. Practical Example
**Product-catalog cache (assumptions):** 5M products, 120B JSON per product, traffic concentrated on 800k SKUs (16% hot).
- Payload: 800k × 120 B ≈ 96 MB. Structure/resident overhead ~1.5x → ~145 MB. Add one local cache per 20 app nodes of the same 10k top SKUs → 20 × (10k × 120 B × 2) ≈ 48 MB more.
- Total ≈ ~200 MB shared + locals — comfortably one 2 GB Redis. The *remaining* 4.2M cold SKUs are the trap: tempting to cache, worthless for hit ratio, and they'd 5x the node count for nothing.

## 10. Scaling
- **One node fits:** size it, aim for 60-80% RAM max so the allocator/GC and failover don't OOM.
- **One node doesn't fit:** shard the keyspace with distinct key-partitions or [[consistent-hashing|Consistent Hashing]]; node count = effective bytes ÷ per-node usable RAM.
- **Local vs distributed:** local caches duplicate the hot set per node (cost × node count) but for the top-K identical objects that's cheaper than a network round trip; shared caches store once and win on set size.
- **Hot keys cap throughput before size does:** a 100 KB super-key can hammer one shard even when RAM is ample — sizing must report the *biggest key* separately from the *total bytes*.

## 11. Reliability and Failure Scenarios

| Failure | What Happens | Detection | Recovery | Trade-off |
|---------|--------------|-----------|----------|-----------|
| Undersized + peak | Hit ratio collapses, DB storm | Miss ratio spike at peak | Raise nodes/set, warm | cost |
| Node OOM/failover | Shards lose keys to rebalance, misses | Shard health | Replica per shard | memory × 2 |
| Value-outgrows-slab | Memcached slabs waste RAM | Item count vs memory | Tune slab sizes | tuning effort |
| Wrong cardinality | Packed but useless cache slots | Huge key count, low ratio | Re-key hot set only | design time |

## 12. Consistency and Correctness
Cache size and consistency interact: a longer TTL lets a larger set stay cached (fewer refills) at the cost of staleness; write-invalidation shrinks effective freshness windows but does not change *bytes*. Size assumes the cache is disposable — the design must still state that a cache-miss path to the DB is always viable, so the RAM number is a cost, not a correctness guarantee.

## 13. Performance
Size buys hit ratio, hit ratio buys latency: an undersized cache routinely wastes 90% of the latency benefit caching exists for. Overhead is not free — Redis is single-threaded, so a 5x memory overhead (structure-heavy keys) also slows operations; choose key encoding (int keys, hashes vs strings) as part of the same estimate.

## 14. Security
- Caches at the edge or in shared pools must not store PII/secrets in uncounted plaintext slots; sizing must include per-tenant isolation space so one tenant's keys can't evict another's (see [[tenancy-and-cells|Multi-Tenancy and Cell-Based Architecture]]).
- Allowlist key inputs — a cache whose keys are attacker-controlled can be filled until it evicts legitimate data.

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| Cache only hot set | Small RAM, high ratio | Cold-start misses, misses for long tail | Most products |
| Cache entire keyspace | No cold-miss on any key | RAM × keyspace, low ROI | Small datasets |
| Shared cache | One copy, easy sizing | Network hop, node SPOF-ish | Moderate-large set |
| Local caches | Zero-latency hot hits | N copies of hot set | Small identical hot set |

## 16. Common Mistakes
- Sizing to the whole dataset ("we have 5M products — cache them all") instead of the hot set.
- Forgetting structure/resident overhead and quoting pure payload bytes.
- Adding replicas without multiplying the copy by the replica count.
- Making the cache 100% full at design time — no headroom for churn and failover.
- Ignoring the largest-key outlier that can melt a shard even when total bytes are fine.

## 17. HLD vs LLD Boundary
HLD: hot-set cardinality, value sizes, overhead factor, replication, node count, eviction policy. LLD: picking the exact Redis data type/encoding, tuning slab classes, choosing a serializer, writing the key format that keeps bytes down.

## 18. Interview Questions

### Beginner
- What is the difference between "dataset size" and "cache size"?
- A cached record is 1 KB online. Why might it cost more RAM than that?

### Intermediate
- Size a cache for 2M user profiles at 500 B each where 30% are hot, with structure overhead and one replica.
- When does a local-per-node cache cost more total RAM than one shared cache?

### Advanced
- Your product caches 200M feed items and the hit ratio is only 35%. Diagnose with the size model and fix.
- Design a cache where a single 5 MB key must not pin a shard over its RAM budget.

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- Size the hot working set, not the whole dataset.
- Cache bytes = hot keys × value bytes × overhead (1.2-2x) × replicas.
- Overhead (dict/headers/slabs) is real RAM, include it.
- Add 25-50% headroom and leave room for N-1 failover.
- Report the largest key separately — hot keys cap throughput before size does.

### 30-Second Explanation

Find the distinct keys the traffic actually touches, multiply by their average value size, inflate by structure overhead (1.2-2x), add the replication factor (1 for a shared cache, node-count for local caches), and add 25-50% headroom to get effective cache RAM. Then sanity-check: the hot set should dominate the hit ratio, and the biggest single key must not pin a shard.

### Interview Traps

- Quoting dataset size instead of working-set size.
- Pure payload bytes with no overhead factor.
- Forgetting local caches duplicate the hot set per node.
- Sizing to 100% RAM with no headroom or failover space.
- Ignoring the super-key that melts one shard.

### Key Trade-Off

Cache size trades RAM for latency and hit ratio: big enough to hold the hot set at high ratio, small enough to keep cost and node count sane — the exact balances are the hot-set cardinality you measure and the overhead you did not forget.

## 20. Related Concepts

### Prerequisites

- [[capacity-estimation|Capacity Estimation]] — the QPS and read-ratio inputs that reveal the hot set.
- [[read-write-ratio|Read/Write Ratio]] — read-heavy workloads are the ones worth sizing a cache for.

### Commonly Used Together

- [[caching|Caching]] — hit ratio, TTL, eviction, and invalidation are the policy this size feeds.
- [[memory-estimation|Memory Estimation]] — cache RAM is memory estimation applied to an object store.
- [[replication-overhead|Replication Overhead]] — the replica multiplier in the cache-size formula.
- [[consistent-hashing|Consistent Hashing]] — sharding a cache that outgrows one node.

### Advanced Concepts

- [[redis|Redis]] and [[memcached|Memcached]] — the concrete stores whose overheads the estimate encodes.

Related planned topics (not authored yet): cache profile/usage telemetry, key-format budget.

## 21. References
Redis and Memcached memory/overhead documentation (value-size limits, slab behavior) provide the resident-overhead constants; standard cache-sizing material in HLD sources (Alex Xu, Grokking). Verify overheads with the store you actually deploy.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- Why size the hot working set instead of the whole dataset?
> Cache RAM buys hit ratio, and hit ratio comes from the fraction of keys traffic actually touches. Caching 5M products when only 800k are requested inflates RAM 6x for no extra hits and may, worse, evict the hot keys constantly (thrash) that a smaller, carefully-bounded cache would serve.

> [!question]- 2M profiles, 30% hot, 500 B each, overhead 1.5x, one replica. Cache bytes?
> Hot set = 2M × 30% = 600k profiles. Payload = 600k × 500 B ≈ 300 MB; with overhead → 450 MB; with one replica → 900 MB; add ~30% headroom → ~1.2 GB, near one modest Redis. State each multiplier so the interviewer can check the chain.

> [!question]- What does structure overhead mean and why does it multiply RAM?
> Beyond the value bytes, the store keeps keys, dict/collection headers, per-entry pointers, allocator waste and (in Memcached) slab rounding. That adds roughly 20-100% to the payload figure, so a 500 B JSON record can cost 750 B-1 KB resident. Quotes in "pure payload" quietly under-size.

> [!question]- Trade-off: shared vs local cache for RAM and latency.
> A local cache duplicates the hot set on every node — total RAM = hot set × node count, but hits cost zero network and are 10-100x faster. A shared cache stores the set once and is memory-efficient at scale, but adds a hop and a possible SPOF. Choose local only when the hot set is small and identical across nodes.

> [!question]- Failure scenario: 200M feed items cached, hit ratio 35%. Diagnose.
> The model is probably caching cardinality, not hotness — likely too many cold keys filling slots while true hot keys get evicted, plus oversize overhead, plus a mispriced replica count. Fix: re-scan traffic for the top-K keys of the real working set, bound the cache to that, and shrink key/value encodings.

> [!question]- Design decision: a 5 MB super-key versus total 200 MB of small keys.
> Total bytes say "one small node"; the super-key says "hot shard". A single large key serializes on the Redis instance holding it and can pin a shard's RAM budget during replication. Size like a cluster: report max key size, count large keys, and partition explicitly so one key can't straddle-do the shard.

> [!question]- Why add 25-50% headroom and N-1 space?
> Working set grows with usage shifts; eviction churn needs slack; and under a node failure the peers inherit that node's keys, which means the fleet must hold all data at 60-75% rather than 100%. Without headroom the cache survives only the traffic you predicted, which never happens.

> [!question]- Interview scenario: the interviewer says "just cache everything, it's only 5 GB." Respond.
> Ask what fraction of traffic the 5 GB realizes — if the hot set is 1 GB, a 5 GB cache buys nothing but cost and cold-key thrash. Present the working-set math, state the overhead and replica multipliers, and recommend caching the true hot set at a defined hit ratio instead.

## 23. When Should I Use This?

### Use it when

- You are adding [[caching|Caching]] and need node/RAM sizing.
- You must choose shared vs local cache placement.
- You are deciding whether caching is even worth the RAM for a workload.

### Avoid it when

- The real problem is write volume (caching writes is usually wrong — see [[read-write-ratio|Read/Write Ratio]]).
- The workload has no hot set (uniform random reads) — memory spent on a cache buys nothing.
- You need a measured profile rather than an estimate — instrument first, then size.

### What problem does it solve?

Problem: cache hits are gated by RAM, and teams either undersize (cache thrash, DB storms) or oversize (wasted cost). Solution: a formula that maps the hot working set — keys, value bytes, structure overhead, replication — to effective RAM and node count, so the cache holds the keys that create the hit ratio it was bought for.

### What problem does it NOT solve?

It does not decide TTL/invalidation policy (that's [[caching|Caching]]), does not protect you from a miss-storm the moment the cache clears (a sizing issue only in the surface sense — the miss path must always survive), and its constants stay guesses until your real key sizes and cardinality are measured.

## 24. Decision Connections

Decisions that go together with cache size estimation:

- [[capacity-estimation|Capacity Estimation]] — the QPS/hot-set discovery that feeds cardinality.
- [[read-write-ratio|Read/Write Ratio]] — read-heavy workloads justify the RAM spend.
- [[caching|Caching]] — the policy (TTL, eviction, write-strategy) the size serves.
- [[memory-estimation|Memory Estimation]] — the sibling estimate for node memory and churn.
- [[replication-overhead|Replication Overhead]] — the replica multiplier every copy pays.
- [[consistent-hashing|Consistent Hashing]] — sharding the cache when the set exceeds one node.
- [[redis|Redis]] / [[memcached|Memcached]] — the concrete overhead constants to plug in.

Decision tree:

```
Will caching pay off?
    |
    +-- Read-heavy with a real hot set?
    |      → [[cache-size-estimation|Cache Size Estimation]]
    |         |
    |         +-- Hot set fits one node?      → single Redis/Memcached node
    |         +-- Larger than one node?       → [[consistent-hashing|Consistent Hashing]] sharding
    |         +-- Identical top-K per node?   → local caches instead
    |
    +-- No hot set (uniform reads)?
    |      → skip; apply [[database-replication|Database Replication]] instead
    |
    +-- Write-bound?
           → invest in the write path, not cache RAM
```