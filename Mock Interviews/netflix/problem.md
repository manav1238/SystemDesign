---
title: "Design Netflix (study transcript)"
status: active
tags: [hld, mock, netflix]
---

# Design Netflix — Problem Statement

## Problem Statement

Design a global video-streaming service similar to Netflix.

**Core product surface:**

- Subscribers browse a catalog of films, television series, and documentaries, and watch them on TVs, phones, tablets, and web browsers.
- Every device plays video smoothly over an unreliable network, adapting the video quality to the connection in real time.
- The service maintains per-user state: viewing history, continue-watching progress, profiles, preferences, parental controls, and personalized rows and artwork.
- The service is available in many countries, and the catalog is different in every country because content rights are licensed per territory and per time window.
- The business runs two tiers: an ad-supported tier and a standard ad-free tier, with a per-device concurrency limit.
- Netflix measures its own quality: startup time, rebuffer ratio, and playback failures, per device and per network.
- Playback and interaction telemetry drives a large offline analytics platform and a large A/B testing platform.

**The interviewer says:**

> "Assume roughly 300 million paid subscribers worldwide, each watching about 2 hours a day. Peak concurrency runs about 1.4 times the daily average. Design the system so that playback starts fast, video never stalls, the service survives the loss of a region, and the content catalog is correct per territory. Explain your bandwidth math, your storage math, and where the money is. Also explain why you would or would not run your own CDN."

## Clarifying Questions You Should Ask

Ask these before you draw a single box. The licensing and territory questions are the ones that distinguish a prepared candidate.

### Product scope and business model

1. Is this subscription-only, or do we need the ad tier with ad decisioning, ad stitching into the manifest, and impression reporting?
2. Do we need profiles, PINs, and parental controls, and do they apply to video or also to account actions?
3. Are there device concurrency limits, such as 4 screens at once, and do they need to be strictly enforced?
4. Do we need downloads for offline viewing, which means DRM keys, license servers, and device trust?
5. Is live content in scope? Sports and news change the delivery guarantees and the manifest cadence.
6. Is a "My Netflix" row, where subscribers add their own titles to the catalog, in scope? It changes the catalog from a global read-only asset into a per-user mutable collection.
7. Is data analytics for the business in scope, or only the streaming-serving system?

### Ingest and content

8. How does content arrive? I am assuming studio masters delivered to us, not user uploads. Is that right?
9. Do we encode, or does content arrive pre-encoded? Do we handle the encoding, and is that in scope?
10. Does the catalog grow by thousands of titles a day at a drop, or by a handful a day normally? This decides whether encoding is a real pipeline or a batch job.
11. Do we need subtitles, audio description, and multiple audio tracks per title, and do they need to be per-language?
12. What is the maximum quality target per tier, and is 4K plus HDR in scope?

### Territory and licensing

13. Are licenses per-territory, per-time-window, and sometimes per-language? I am assuming yes, and it is the single hardest part of the catalog.
14. Are there data-residency requirements for viewing data in any jurisdiction?
15. Is geo-blocking legally required in any market, meaning a subscriber in country A must not be able to play country B's content?
16. Which territories are launch markets, and how many regions does that imply?
17. Can a single account be used in multiple countries at once, as with travel?

### Non-functional requirements

18. What is the availability target, and do we differentiate between catalog browsing and playback?
19. What is the startup-latency budget, time from pressing play to first frame, and do we differentiate by device class? A TV is a different problem from a phone.
20. What is the rebuffering budget, expressed as a percentage of total watch time? This is the metric users actually judge us on.
21. How stale can viewing history and continue-watching progress be? I expect eventual, with a read-your-write path for the user's own row.
22. Do personalized rows and artwork have a freshness target, and how much staleness is acceptable before it looks broken?
23. Is the A/B testing platform in scope? Thousands of concurrent experiments with no interaction with the serving path is a real constraint.
24. Is there a cost ceiling, and is egress treated as the primary cost driver?

### Constraints and preferences

25. Are we buying CDN capacity from third parties, running our own, or both? I want to argue for a hybrid and explain why.
26. Do we control the player application on every device, and do we control the client SDK? This decides how much playback logic we own versus the CDN's.
27. Are we on a single cloud provider, or are we multi-cloud?
28. What is the priority when we must trade off: startup latency, rebuffering, personalization quality, catalog correctness, or cost?

## What You Are Evaluated On

The interview is scored on five phases, matching [[06-hld-interview-checklist|HLD Interview Checklist]].

### Phase 1 — Requirements Clarification

Did you probe product scope, ingest model, territory licensing, and device classes before designing? Did you recognize that "the catalog" is a per-territory, time-bounded data problem rather than a list? Did you state availability, startup-latency, and rebuffering targets as measurable SLIs rather than saying "fast"? See [[functional-vs-non-functional-requirements|Functional vs Non-Functional Requirements]].

### Phase 2 — Scale Estimation

Did you estimate concurrency from watch time, then derive bandwidth, then derive request counts from segment duration? Did you separate the three request classes, play starts, segment fetches, and progress heartbeats, because they differ by two orders of magnitude and have different designs? Did you note that this is a bandwidth business and not a storage business, in stark contrast to a user-upload platform? See [[capacity-estimation|Capacity Estimation]], [[server-capacity|Server Capacity]], and [[latency-budget|Latency Budget]].

### Phase 3 — High-Level Architecture

Did you draw a device-facing BFF layer distinct from internal services? Did you place Open Connect or an equivalent edge appliance tier explicitly, with its own fill and routing logic, rather than writing "CDN"? Did you model the per-title adaptive-bitrate ladder as a per-title artifact rather than a global constant? See [[edge-computing|Edge Computing]] and [[backend-for-frontend|Backend for Frontend]].

### Phase 4 — Deep Dive

Did you go deep on at least four of: per-title encoding optimization, the edge appliance fleet and its fill and routing policy, the territory-scoped catalog with effective-dated licensing, watch progress and continue-watching at scale, the two-dimensional recommendation model, the telemetry and analytics pipeline, A/B testing isolation, and multi-region failure handling? Did you reason about consistency per operation, especially for the user's own row? See [[strong-vs-eventual-consistency|Strong vs Eventual Consistency]] and [[geo-dns-anycast|Geo-DNS and Anycast]].

### Phase 5 — Trade-offs and Failure Scenarios

Did you name the cost of every decision, especially the self-built CDN versus the hybrid? Did you walk through concrete failures: primary DB death, a season drop causing a hot title, a cold cache after regional failover, replica lag on the user's own progress, an unavailable territory license, and a stalled encoding pipeline? Did you close with an explicit statement of where the money goes and why the design follows from that? See [[trade-off-analysis|Trade-Off Analysis]] and [[graceful-degradation|Graceful Degradation]].

## Related Reading

- [[06-hld-interview-checklist|HLD Interview Checklist]] — the skeleton this problem is scored against
- [[01-rapid-revision|Rapid Revision]] — one-liners per concept for the day before
- [[media-processing|Media Processing]] — the per-title encoding ladder
- [[edge-computing|Edge Computing]] — the Open Connect tier
- [[data-residency|Data Residency]] — territory and licensing constraints
