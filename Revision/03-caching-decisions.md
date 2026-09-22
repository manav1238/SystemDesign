---
title: Caching Decisions
status: active
tags:
  - hld
  - revision
  - caching
---

# Caching Decisions

Rules of thumb for caching questions.

## 1. Should I cache at all?

- Same data read many times, recompute/query is expensive, slight staleness is fine → cache → [[caching|Caching]]
- Data changes constantly or per-user secret → be careful; keep TTL short or skip.
- Hot read-heavy single source → edge+local+distributed before touching your database review: [[load-balancing|Load Balancing]], [[reverse-proxy|Reverse Proxy]], [[cdn|CDN]].

## 2. Which layer?

- Static assets, global users → [[cdn|CDN]]
- Repeated backend responses, TLS/compression offload → reverse proxy → [[reverse-proxy|Reverse Proxy]]
- App-level hot data (user profiles, feed) → local cache + [[caching|Caching]] (distributed Redis-style tier)

## 3. Which write strategy?

- Simple, best hit rate for read-heavy → cache-aside (read-through/write-around)
- Must never serve stale, consistency critical → write-through
- Tolerate async flush for throughput → write-back (accept risk of loss)

## 4. What breaks caches?

- Invalidated content still served → version keys / TTL
- Lots of requests on one cold key → stampede; add jitter, locks, single-flight
- Requests for data that doesn't exist → penetration; negative caching, bloom filter, validate early
- Cache tier churns at once → avalanche; randomize TTLs, prewarm
- These four failure modes are the core of [[caching|Caching]] — always name them.

## Decision tree

```
Need to cache?
  Yes → which layer? static → [[cdn|CDN]]
                        backend repeated → [[reverse-proxy|Reverse Proxy]]
                        app hot data   → [[caching|Caching]] (distributed tier)
  Consistency critical? → write-through / short TTL
  Reads ≫ writes?       → cache-aside
```