---
title: "Design a URL Shortener (study transcript)"
status: active
tags: [hld, mock, url-shortener]
---

# Design a URL Shortener — Problem Statement

## Problem Statement

Design a URL shortening and link-management service comparable to Bitly.

**Core product surface:**

- Any user (anonymous or authenticated) can submit a long URL and get back a short URL of the form `https://sho.rt/aB3xY9z`.
- Anyone who visits a short URL is redirected to the original destination.
- An authenticated owner can optionally request a custom alias instead of a random code.
- Every click is counted, and the owner can view per-day click analytics for their links.
- Links can have an optional expiry date, and can be deleted or deactivated by the owner.
- The system must be cheap, because the vast majority of traffic is anonymous reads of codes that already exist.

**The interviewer says:**

> "Assume 100 million URLs are stored. New URLs are created about 1 million times a day. A stored URL gets resolved roughly 10 times per day on average, so the read-to-write ratio is about 10 to 1. Peak traffic is roughly 10 times the daily average and is heavily concentrated on a small number of viral links. Design the system. I want to know how you generate short codes, what your source of truth is, how you keep redirect latency low, how you count clicks without slowing down redirects, how you shard, and what happens when your primary database dies or one link gets a million clicks in a minute."

## Clarifying Questions You Should Ask

Ask these before you draw a single box. Each one signals that you are scoping the problem deliberately rather than guessing.

### Product scope

1. Do links need a user account, or is anonymous shortening allowed? Anonymous means abuse, spam, and takedown requests, and it changes the create-path rate limit.
2. Is the short URL served on a dedicated domain, or on a path of an existing domain? A dedicated domain means DNS and certificate management plus a clean, cacheable 302.
3. Do we need custom aliases, and if so are they unique globally or only within a user's namespace? This single question decides whether the shard key is the short code or the user.
4. Can a link's destination be edited after creation? Editing breaks historical click attribution and cached copies.
5. Do we need link-level access control (private links with tokens), or are all short URLs public by default?
6. Is the analytics product a per-day click count only, or do we need referrer, country, device, and timestamp breakdowns? This decides whether analytics is a rollup table or a full event pipeline.
7. Do we need bulk creation (CSV upload) or an API that other products call server-to-server with a high volume cap?

### Scale and traffic

8. What are the monthly and daily active users? If not given, I will propose numbers and say so out loud.
9. What is the read-to-write ratio? I expect roughly 10 to 1 and want it confirmed, because it decides cache and database investment.
10. What is the average length of the long URL? Real URLs with tracking parameters run 300 to 800 bytes, and that dominates row size.
11. What is the expected distribution of clicks across links? I expect a heavy long tail with a very fat head. How fat is the head? What is the p99 clicks per link per minute?
12. Is the traffic global, or is it regional? If global, a short code has to resolve from anywhere, which means the hot path is latency-sensitive and cache-heavy.
13. Do we need a preview or safety-check service (unfurl title and favicon, malware and phishing screening) on the create path? That adds synchronous third-party latency to writes.

### Non-functional requirements

14. What availability target do we want for redirects? I would propose 99.99 percent, because a dead short link is a dead marketing campaign.
15. What is the redirect latency budget? I intend to argue for a p99 under 100 milliseconds end to end, and I need to know how much of that is TLS, edge, and hop count.
16. How fresh must the click count be? I intend to propose a few seconds, and argue that exact real-time counts are not worth blocking a redirect for.
17. What is the acceptable staleness on the mapping itself for a brand-new link? A link created 200 milliseconds ago that 404s because a read replica is behind is a visible product bug.
18. Are there legal constraints on analytics data, for example right-to-delete or link-expiry guarantees? I plan to design expiry as a hard guarantee, not a best-effort sweep.
19. Do we need to survive loss of an entire region, or is multi-AZ within one region enough for the first version?

### Constraints and preferences

20. Which cloud or platform are we on? Managed MySQL versus a hand-rolled store changes the sharding answer completely.
21. Is a cache such as [[redis|Redis]] allowed on the hot path, or must redirects work with the cache fully cold? Cold-cache correctness is the real availability test.
22. Are we okay with an eventually consistent click count, and what is the acceptable data loss window if the analytics pipeline is down?
23. Do we need abuse prevention on create, given that the service is trivially usable for phishing and malware distribution?
24. What is the priority order when forced to trade off: redirect availability, redirect latency, analytics accuracy, or create availability?

## What You Are Evaluated On

The interview is scored on five phases, matching [[06-hld-interview-checklist|HLD Interview Checklist]].

### Phase 1 — Requirements Clarification

Did you ask scoping questions before estimating, or did you start designing immediately? Did you separate functional from non-functional requirements explicitly? Did you surface assumptions out loud instead of silently choosing numbers? The two highest-signal questions here are custom-alias uniqueness semantics and how fresh click counts must be, because they change the data model. See [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]].

### Phase 2 — Scale Estimation

Did you show arithmetic with real numbers for writes per second, redirect QPS, storage, and analytics volume? Did you notice that peak is a multiple of the daily average and that traffic concentrates on a handful of keys, which is the single most important fact about this problem? Did you size storage from the long-URL length rather than from a row count alone? See [[capacity-estimation|Capacity Estimation]], [[read-write-ratio|Read/Write Ratio]], and [[dau-mau|DAU/MAU]].

### Phase 3 — High-Level Architecture

Did you draw a clear diagram with named services? Did you identify that the create path and the redirect path are two different systems with completely different latency budgets, and design them separately? Did you put [[cdn|CDN]] and [[load-balancing|Load Balancing]] in front of redirects, [[caching|Caching]] as a first-class component, and a database as the source of truth rather than the read path? See [[control-plane-vs-data-plane|Control Plane vs Data Plane]] and [[stateless-vs-stateful-services|Stateless vs Stateful Services]].

### Phase 4 — Deep Dive

Did you go deep on at least four of: base62 short-code generation and collision math, pre-allocated key pools, the source-of-truth decision, cache-aside plus write-through, 301 versus 302 semantics, the asynchronous click analytics pipeline, [[sharding|Sharding]] by short code, [[shard-key|shard key]] selection, [[database-replication|Database Replication]] with synchronous writes, a Bloom filter for invalid codes, TTL and expiry cleanup, and [[probabilistic-data-structures|probabilistic data structures]]. Did you reason about consistency per data type rather than one global model? See [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] and [[idempotency|Idempotency]].

### Phase 5 — Trade-offs and Failure Scenarios

Did you name the cost of every major decision? Did you walk through concrete failures: primary database death, a viral link creating a single hot key, a cache stampede when a hot key expires, [[replication-lag|replica lag]] making a brand-new link 404, the analytics broker being down, and traffic doubling overnight? Did you close with a coherent summary of what breaks first and how the system degrades? See [[graceful-degradation|Graceful Degradation]], [[trade-off-analysis|Trade-Off Analysis]], and [[hotspot-handling|Hotspot Handling]].

## Related Reading

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the skeleton this problem is scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before
- [[caching|Caching]] — the component that actually carries this system
- [[sharding|Sharding]] — the decision that is hardest to reverse
