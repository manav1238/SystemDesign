---
title: Network Latency / Bandwidth / Packet Loss
category: Networking
priority: must-know
status: learning
difficulty: medium
interview_ready: false
tags:
  - hld
  - networking
  - performance
---

# Network Latency, Bandwidth, Packet Loss

## 1. One-Line Definition
Network latency is the time a packet takes to cross a network, bandwidth is how many bits the path can carry per second, and packet loss is how many of those packets never arrive — the three physical parameters that set the floor on every distributed system's throughput and user experience.

## 2. Why Do We Need It?
Every design decision — where to put replicas, whether to use [[caching|Caching]], how to size machines, whether one region is enough — is secretly a latency, bandwidth, and loss decision. Calibrate them wrong and you budget impossible timeouts, undersize the network, and build retries that trigger on physics rather than on real failures. Interviewers pair these numbers directly with [[latency-vs-throughput|Latency vs Throughput]] and [[capacity-estimation|Capacity Estimation]].

## 3. Simple Intuition
A highway: **latency** is the drive time (distance plus traffic lights), **bandwidth** is the lane count and cars per minute, **packet loss** is crashes and potholes. A wide highway cannot shorten a 500 km drive — latency is not cured by bandwidth, and no number of lanes helps if every mile has a pileup.

## 4. What Happens Without It?
Teams "add bandwidth" to fix a latency problem, or set a 200 ms timeout because the localhost test was 2 ms, then wonder why production times out. Every cross-region write, CDN miss, and fan-out gets silently calibrated against wrong physics — measured outages await.

## 5. Core Idea
- **Latency is a sum, not one number:** propagation (distance divided by speed of light in fiber), transmission (packet size divided by bandwidth), queueing (waiting at routers), and processing (server CPU and NICs). RTT = round trip.
- **Physics wins:** light in fiber travels ~200,000 km/s, about 5 us/km one way and 10 us/km round trip. Same-region RTT is 1-5 ms; cross-continent is 60-170 ms. No protocol shortens distance.
- **Bandwidth and latency compose into throughput:** only about `bandwidth x RTT` bits can be in flight at once; a single [[tcp|TCP]] flow tops out near `window / RTT`.
- **Loss is the multiplier:** TCP retransmits lost bytes, so 1% loss can cut effective throughput by ten to fifty times and inflate the tail dramatically.
- **Design to the tail:** queueing makes p99 much worse than p50 — "average is fine" hides real latency failures ([[tail-latency|Predictable Tail Latency]]).

## 6. Important Terminology

| Term | Simple Meaning |
|------|----------------|
| RTT | Round-trip time for a packet |
| Propagation delay | Light covering the distance |
| Transmission delay | Packet size divided by bandwidth |
| Queueing delay | Waiting in router buffers |
| Jitter | Variance in latency between packets |
| Bandwidth | Bits per second the path sustains |
| Packet loss | Fraction of packets dropped |
| Bandwidth-delay product | In-flight bytes = bandwidth x RTT |
| Goodput | Application-useful throughput after retries |
| Tail latency | The slowest percentiles, not the average |

## 7. Basic Architecture

```mermaid
flowchart LR
    U[User device] -->|RTT 1 to CDN edge| E[CDN edge]
    E -->|RTT 2 to origin| O[Origin server]
    O --> D[(Database same region)]
```

Each arrow is one round trip, and each hop adds queueing under load. The latency budget is the sum of every arrow; a CDN cache removes most of the far-arrow traffic.

## 8. Request or Data Flow
1. DNS lookup (resolution plus TTL-bounded caching) — tens of ms once, near zero when cached.
2. TCP and TLS handshake — 1-2 RTTs (see [[http-and-https|HTTP and HTTPS]]).
3. Round trip to the edge/CDN, then possibly to origin — one extra RTT per cache miss.
4. The database read/write adds its own RTT, which is why cross-region writes are expensive.
5. Under loss, each retransmission adds an RTT — the reason tail latency explodes.

Every step is a budget item you can trace in [[distributed-tracing|Distributed Tracing]].

## 9. Practical Example
**Global API with an edge cache (assumptions):** 100M requests/day; 60% edge cache hits.
- Same-region CDN hit is about 10 ms; edge miss plus origin about 25 ms; a DB write adds about 5 ms.
- Effective p50 is about 0.6 x 10 + 0.4 x 25 = 16 ms.
- At 0.5% mobile loss, retransmissions push p99 to 400-600 ms — "average is fine," the p99 is not.
- Capacity sanity: 10 Gbps over 100 ms RTT allows only about 125 MB unacknowledged in flight — one TCP flow cannot go faster without a bigger window or parallelism.

## 10. Scaling
- **Distance is invariant:** more users worldwide does not shrink cross-continent RTT — you place copies closer via [[cdn|CDN]], [[edge-computing|Edge Computing]], and [[geo-dns-anycast|Geo-DNS and Anycast]].
- **Bandwidth scales with money and parallelism:** aggregate many flows and NICs; a single TCP flow stays window-over-RTT capped.
- **Queueing is superlinear:** near 100% utilization, queueing delay explodes — keep headroom on hot paths.
- **Loss scales with congestion and reroutes:** recover with more edge buffers and anycast, never by targeting "lower latency."

## 11. Reliability and Failure Scenarios

| Failure | What Happens | Detection | Recovery | Trade-off |
|---------|--------------|-----------|----------|-----------|
| Link or transit failure | RTT spike, packet loss | Probes, traceroute, SLO alarms | Reroute, [[regional-failover|Regional Failover]] | capacity during reroute |
| Congestion | Queueing plus loss, tail balloons | p99 monitors, drop counters | Throttle, backpressure, cache | some traffic sacrificed |
| Server stall | Processing delay inflates RTT | Server metrics | Auto-scale, queue | cost |
| DDoS | Bandwidth and queue saturation | Interface drop counters | Scrub, anycast, rate-limit | an extra hop |

Monitor **RTT, drop rate, and queue depth as first-class signals**, not just request latency.

## 12. Consistency and Correctness
- High latency makes distributed state observationally stale: every "fresh" remote read pays the RTT, which is why consistency choices are latency choices ([[strong-vs-eventual-consistency|Strong vs Eventual Consistency]]).
- Timeout-driven decisions fail under jitter: an interval safe at p50 is a false partition at p99. Separate timeout design from latency measurement.
- Retries under loss must back off ([[retry-and-timeout|Retry and Timeout]]); the retry storm is the classic correctness bug built on wrong latency assumptions.

## 13. Performance
- **Cut round trips:** combine calls ([[api-composition|API Composition]]), keep-alive, and serve from the nearest cache.
- **Compress, don't just widen:** [[compression|Compression]] and smaller payloads beat wider pipes; honor the bandwidth-delay product when sizing transfers.
- **Make loss not cost latency:** move writes to reliable links, add FEC where freshness rules, and design loss-tolerant delivery ([[udp|UDP]]/QUIC).
- **Benchmark responsibly:** localhost is about 0.1 ms, same rack 0.5 ms, same region 1-5 ms — localhost numbers never describe the fleet.

## 14. Security
- DDoS saturates bandwidth and queues, turning a performance issue into an availability attack — absorb at the edge, throttle, and geo-filter ([[availability|Availability]]).
- Interface drop counters and queue depth distinguish "congested" from "attacked."
- Latency is also a compliance decision: where the edge serves determines jurisdiction ([[data-residency|Data Residency]]).

## 15. Trade-Offs

| Choice | Advantages | Disadvantages | When to Use |
|--------|------------|---------------|-------------|
| More caching / edge | Cuts latency, absorbs load | Staleness and invalidation cost | Read-heavy global |
| Single region | Simplest, strongest consistency | High latency for remote users | Local product |
| Multi-region | Low global latency | Cross-region write and consistency taxes | Global product |
| Sync replication | Fresh reads locally | Write latency = RTT to farthest | Strict-consistency writes |
| Loss-tolerant UDP | Flat latency under loss | Data may be incomplete | Media, games |

## 16. Common Mistakes
- Confusing bandwidth with latency and "adding lanes" to fix distance.
- Designing to p50 while p99 is ten times worse, then chasing "random" timeouts.
- Assuming the datacenter network is lossless — it is about 0.01%, not 0%.
- Setting timeouts from localhost measurements.
- Ignoring the bandwidth-delay product: a "10 Gbps" link delivers far less to a faraway peer.

## 17. HLD vs LLD Boundary
HLD: per-hop latency budgets, cache hit-rate targets, edge and region placement, timeout and retry policy, monitoring signals (RTT, loss, queue). LLD: the exact backoff constants, one service's cache TTL, the packet capture on a single connection.

## 18. Interview Questions

### Beginner
- What is the difference between latency and bandwidth, with an example?
- Why can't you fix a latency problem by adding bandwidth?

### Intermediate
- Estimate latency for a Mumbai user hitting a Virginia server, and justify your number.
- What does packet loss do to TCP throughput, and why is 1% loss worse than it sounds?

### Advanced
- You have 10 Gbps and a 120 ms RTT. Why can't one connection deliver 10 Gbps, and what do you do?
- Design a latency budget for a global web request and a cache strategy that meets a 150 ms p99.

## 19. Interview Memory Summary

> [!abstract]- Interview Memory Summary

### Remember

- Latency = time; bandwidth = bits/sec; loss = drop fraction. Distinct.
- Propagation is about 5 us/km through fiber; cross-continent is 60-170 ms.
- Throughput ceiling is roughly window divided by RTT (bandwidth-delay product).
- Loss collapses throughput: retries multiply the cost.
- Queueing makes p99 far worse than p50 — monitor the tail.
- Reduce round trips; add edges and caches; never trust localhost numbers.
- Timeouts must be set from network RTT, not local tests.

### 30-Second Explanation

Latency is the time a packet needs (distance plus queueing, physics sets the floor), bandwidth is capacity, and loss is the force multiplier that turns a clean pipe into a stalled one at scale. You budget per hop (DNS, TLS, cache, origin), design to p99 not p50, and bring the answer closer to the user with caching and edges, because no amount of bandwidth shortens a faraway round trip. Sizing throughput needs window-over-RTT math, and timeouts are calibrated to real network RTT, never localhost.

### Interview Traps

- Saying "add bandwidth" to fix a latency complaint — the interviewer wants physics, not lanes.
- Designing on p50s; the outage lives in the p99.
- Assuming zero datacenter loss, then explaining "random" drops later.
- Estimating RTT from localhost numbers.

### Key Trade-Off

Physics is not negotiable: lower latency for distant users means storing or serving the answer closer (edge, cache, replica), and higher single-flow throughput means carrying more in flight — both cost money, staleness, or consistency, which is exactly how bandwidth, loss, and caching trade against each other.

## 20. Related Concepts

### Prerequisites

- [[latency-vs-throughput|Latency vs Throughput]] — the fundamentals home of these metrics.
- [[capacity-estimation|Capacity Estimation]] — turning the numbers into system sizing.

### Commonly Used Together

- [[tcp|TCP]] — the transport whose window-over-RTT math converts the numbers into throughput.
- [[udp|UDP]] — the transport you pick when latency and loss budgets demand freshness.
- [[http-and-https|HTTP and HTTPS]] — handshake RTTs inside every web latency budget.
- [[cdn|CDN]] and [[caching|Caching]] — the latency-killers that trade staleness for speed.
- [[geo-dns-anycast|Geo-DNS and Anycast]] — routing users to the closest of several edges.

### Alternatives

- [[edge-computing|Edge Computing]] — moving computation closer instead of only caching the answer.

### Advanced Concepts

- [[tail-latency|Predictable Tail Latency]] — where queueing and loss hide; design against it.
- [[distributed-tracing|Distributed Tracing]] — the tool that shows where the real per-hop budget is spent.

Related planned topics (not authored yet): latency-budget, geo-load-balancing, quic.

## 21. References
Kleppmann, *Designing Data-Intensive Applications* (ch. 1 and ch. 8: latency and network realities). Kurose and Ross, *Computer Networking: A Top-Down Approach* (performance chapter). Google's "Latency Numbers Every Programmer Should Know" (Jeff Dean) for order-of-magnitude benchmarks. Re-check current cloud inter-region latency figures against vendor docs.

## 22. Active Recall

Test yourself before revealing each answer.

> [!question]- In one sentence each: latency, bandwidth, and packet loss?
> Latency is how long a packet takes end to end, bandwidth is how many bits per second the path sustains, and loss is the fraction of packets that never arrive. They are independent physics, not synonyms.

> [!question]- Why does bandwidth not fix a latency problem?
> Latency is dominated by propagation (distance through fiber at the speed of light) plus queueing; bandwidth only shrinks the transmission term. A 500 km path has the same minimum RTT whether you have 1 Mbps or 100 Gbps — the answer to distance is proximity (edge, cache, replica), not more lanes.

> [!question]- How do bandwidth and RTT together cap a single TCP connection's throughput?
> Only about bandwidth times RTT bytes can be unacknowledged in flight, so maximum throughput is roughly window divided by RTT. At 100 ms RTT and a 1 MB window you cap near 80 Mbps regardless of a 10 Gbps link; you raise the ceiling with a bigger window or parallel connections, never by wishful bandwidth.

> [!question]- Why is 1% packet loss catastrophic, not merely 1% slower?
> TCP treats each loss as congestion: it retransmits and shrinks its window, taking many RTTs to recover. Because that penalty multiplies across every affected flow, 1% loss can cut effective throughput ten to fifty times and blow out tail latency — a single lost packet stalls an entire pipeline behind it.

> [!question]- Your p50 is 15 ms but users complain of "hangs." What is the likely culprit, and what do you measure?
> The tail: queueing under load makes p99 several times the p50, and any loss throws retransmission stalls on top. Measure p99 and p999 latency, drop rate, and queue depth — not just the average — and look for the one hot hop (origin, DB, TLS) that owns most of the tail.

> [!question]- Trade-off: single region vs multi-region for a global product?
> Single region gives simple, strongest consistency but adds 100+ ms RTT for faraway users; multi-region cuts that latency with local replicas but adds cross-region write taxes, conflict resolution, and consistency complexity. The deciding factor is usually the write path plus the latency budget, not the read path, which caching already fixes.

> [!question]- Interview scenario: meet a 150 ms p99 global budget. Where do the milliseconds go?
> Budget by hop: DNS (cached near zero, first about 20 ms), TLS (edge reuse saves an RTT), edge hit 10-20 ms, edge miss plus origin 30-80 ms. The strategy: a high edge cache hit rate, keep-alive to avoid re-handshakes, same-region reads, and hard timeouts with backoff so tail-causing retries never stack. If the tail still busts, it is loss — fix the transport, not the budget.

## 23. When Should I Use This?

### Use it when

- You set latency budgets for a distributed system and size timeouts per hop.
- You estimate storage, request, and bandwidth needs with [[capacity-estimation|Capacity Estimation]].
- You place replicas, caches, or edges and must decide what wins: distance, consistency, or staleness.
- You reason about throughput needs as more than raw bandwidth.

### Avoid it when

- The design is entirely at the business-logic layer with no network path to reason about.
- A managed platform owns the edge and you only consume its latency and SLO promises.
- Your product is single-datacenter with captive users; the numbers still apply but the decisions are trivialized.

### What problem does it solve?

Problem: systems are sized and timeouts are set without knowing what the network can physically deliver. Bottleneck: distance, queueing, and loss confound every "it should be fast" guess. Solution: explicit latency, bandwidth, and loss budgets, tail-aware monitoring, and proximity tricks (caching, edges, geography) that make the numbers a design input rather than a post-mortem.

### What problem does it NOT solve?

The metrics do not identify which hop broke (that is tracing), do not fix root causes (a queue is often app-side backpressure, not "the network"), do not remove consistency trade-offs, and cannot make a lossy wireless link reliable without a transport change. They tell you the budget; you still design the system that respects it.

## 24. Decision Connections

Decisions that go together with network performance:

- [[latency-vs-throughput|Latency vs Throughput]] — the metric pair these three numbers belong to.
- [[capacity-estimation|Capacity Estimation]] — turning RTT, bandwidth, and QPS into system sizing.
- [[tcp|TCP]] and [[udp|UDP]] — the transport choice decides whether loss costs a stall or a skip.
- [[http-and-https|HTTP and HTTPS]] — handshake and keep-alive policy inside every latency budget.
- [[cdn|CDN]] and [[caching|Caching]] — the primary levers that trade staleness for latency.
- [[geo-dns-anycast|Geo-DNS and Anycast]] — proximity routing as a latency decision.
- [[tail-latency|Predictable Tail Latency]] — the design target your budget must respect.
- [[retry-and-timeout|Retry and Timeout]] — the policy that must be calibrated to these numbers.

Decision tree:

```
System feels slow or is being sized
    |
    +-- Reads dominate and data is far?
    |      → [[cdn|CDN]], [[caching|Caching]], [[edge-computing|Edge Computing]]
    |
    +-- Writes cross long distances?
    |      → move data closer or accept the write RTT, or [[multi-region-models|Multi-Region Models]]
    |
    +-- Throughput below bandwidth hopes?
    |      → check window divided by RTT and the [[tcp|TCP]] congestion story
    |
    +-- Tail latency the complaint?
    |      → [[tail-latency|Predictable Tail Latency]], queue depth, loss, backoff retries
    |
    +-- Loss the real sickness?
           → reliable transport or [[udp|UDP]]/QUIC depending on freshness needs
```