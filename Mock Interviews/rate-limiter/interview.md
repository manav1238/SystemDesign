---
title: "Design a Rate Limiter — Mock Interview Transcript"
status: active
tags: [hld, mock, rate-limiter]
---

# Design a Rate Limiter — Mock Interview Transcript

Read-only study transcript. Every line is a speaker turn. Reproduce the reasoning, not the words.

## Phase 1 — Requirements Clarification

Interviewer: Thanks for joining. Today we design a distributed rate limiter. We have 200 million registered users, 20 million daily active, and a peak of 300,000 requests per second flowing through the platform. Every API must check the limiter before doing any work. The standard quota is 100 requests per second per user with a burst of 300. Anonymous traffic is limited per IP. Ask me what you need to know.

Candidate: Thank you. I want to understand what the limiter is for before I design it, because the answer changes the failure mode completely.

Interviewer: Go ahead.

Candidate: First, is this limiter an in-process library or sidecar, or is it a network service every call crosses?

Interviewer: Let's start as a shared client library embedded in each service, and I want you to tell me when that stops being enough.

Candidate: Then latency is a non-issue and I control the failure path, which is a big win. I will treat that as the starting point and show you where it breaks.

Interviewer: Second question.

Candidate: What is the limiter for: abuse prevention, protecting a downstream dependency like a database or a third-party API, or cost control?

Interviewer: All three, but abuse prevention is the main one. There was an incident where a single customer's scraper took down the search service for everyone.

Candidate: That tells me the limiter has to be per customer and per endpoint cost, not just per IP, and it tells me abuse is a first-class requirement. Next: do all endpoints cost the same, or is there weighting?

Interviewer: Weighting. A list read costs 1, a search costs 10, a bulk export costs 100. And the limits vary by subscription tier, free versus premium versus enterprise, and enterprise can have a contract number.

Candidate: So the effective rule is a token bucket with a cost, and the limit is a function of tier, customer, and endpoint. Next: do we limit by more than one dimension at once?

Interviewer: Yes. Per user, per IP, per tenant, and per API key, and the request must pass all of them. Plus there is a platform-wide ceiling that is not attached to any customer.

Candidate: So the request consumes from several independent buckets and passes only if all of them pass. That is a fan-out, and it has a latency cost, so I will think hard about how many of those checks actually need to be remote. Next: is the quota global or per region?

Interviewer: Per region for now. A customer's global quota is a future problem.

Candidate: Good, that removes cross-region coordination. Next: how accurate does this need to be? Is it a security control, or is it approximate?

Interviewer: Abusive traffic must be blocked reliably. Legitimate traffic at the boundary can be off by a few percent. It is not a metering system for billing.

Candidate: That licenses an approximate algorithm with bounded error, which is a huge simplification. Next: what happens when the limiter backend is down. Fail open or fail closed?

Interviewer: That is the question I want you to answer, but tell me what you think the options are first.

Candidate: I will come back to it with a nuanced answer, because I do not think one global choice is right. Let me ask two more. How quickly must a policy change take effect?

Interviewer: Within a minute.

Candidate: Then config can be pushed to instances, and it does not need to be read from the database on the request path, which would be a disaster for latency. Last: what is our decision latency budget?

Interviewer: p99 in the low single-digit milliseconds, and a hard timeout of about 5 milliseconds. If the limiter takes longer than that, the platform must not slow down.

Candidate: That timeout is the most important constraint you have given me, and I will treat the limiter as strictly off the critical path. I am ready to estimate.

## Phase 2 — Estimation

Candidate: Traffic first. Three hundred thousand requests per second at peak, every one of which needs a decision. If I do this purely centrally with one network round trip per request, that is 300,000 round trips per second. Let me sanity-check what one request costs.

Interviewer: Go on.

Candidate: A Redis EVAL that does a refill and a decrement is roughly 0.2 to 0.4 milliseconds p50 on a well-tuned node, plus network RTT. A single-threaded Redis node handles on the order of 70,000 to 100,000 such operations per second because Lua scripts are heavier than a plain GET. So 300,000 per second needs at least 300 divided by 70,000, which is about 4.3 shards, and I want headroom for failover, skew, and growth, so I would say 16 to 24 shards, replicated, spread over three availability zones. That is cheap and it is not the interesting part of the problem.

Interviewer: What is the interesting part?

Candidate: State size, because that decides the algorithm. Let me compute both candidates. In any one second, 300,000 distinct requests happen. If the limiter must keep a per-request entry, that is 300,000 entries per second, 18 million per minute, about 1.08 billion per hour. At even 40 bytes per entry in a sorted set or a list, that is 43 gigabytes per hour, and you have to expire and compact all of it. That is the exact sliding window log and it is not viable. So the algorithm must be counter-based or token-based with constant memory per key. That single calculation is the whole reason token bucket wins.

Interviewer: Go back to the key count. How many distinct keys do we actually track?

Candidate: Not every user is active in every second. Twenty million daily actives, and the 300,000 per second peak is concentrated. If the average user makes about 25 requests per session and sessions last a few minutes, the number of distinct users active in a given second is far below 20 million. I will estimate the live key set as 4 to 8 million active keys at peak, which is a deliberately conservative upper bound.

Candidate: Memory: a token bucket per key is a hash or a packed string holding last_refill_timestamp, 8 bytes, and tokens_remaining, 4 bytes, plus the key string of about 20 bytes and Redis overhead, call it 80 to 100 bytes per key. 8 million keys at 100 bytes is 800 megabytes, and sharded over 24 nodes that is about 35 megabytes per node. Trivially cheap. So the memory argument does not even come close to being the constraint, and the design should be chosen on accuracy and behaviour, not on capacity. That is a useful thing to say out loud, because it means I should optimise for correctness, not for a memory budget I do not have.

Interviewer: Good. Now, if we only did it in memory on each instance, what would the effective limit be?

Candidate: With 300 application instances, each enforcing 100 per second locally, the effective limit is 30,000 per second. A single user could send 100 per second to each of 300 instances and get through. So a purely local limiter is wrong by a factor of the instance count, and the entire design problem is closing that gap without paying a network round trip on every request.

Interviewer: And the fixed window boundary problem.

Candidate: A fixed window allows double the limit across a boundary. At 100 per second, a client can send 100 in the last millisecond of one second and 100 in the first millisecond of the next, so 200 in effectively no time. For an abuse control that is a real hole, though a small one.

## Phase 3 — High-Level Architecture

Candidate: Here is the design. The key idea is three tiers, and the tiers exist to solve the 300,000 requests per second problem without a round trip per request.

```mermaid
flowchart TD
    EDGE[Edge: WAF / DDoS scrubbing / IP reputation<br/>coarse volumetric block before anything else]
    LB[L7 Load Balancer<br/>Envoy / nginx / CDN<br/>hard connection cap + overload protection]
    L1[L1 local limiter<br/>leased quota + hot-key guard]
    SA[Service A middleware]
    SB[Service B middleware]
    LIB[shared client lib / sidecar]
    CORE[Limiter core: Go lib / small service<br/>resolve keys, tier lookup + weights<br/>token bucket / GCRA, 5ms hard timeout<br/>circuit breaker]
    RD[Redis Cluster, 24 shards<br/>hash-tagged keys rl:{tier}:{scope}:{id}<br/>TTL = window]
    POL[Policy config store<br/>pushed, 60s propagation]
    DEG[Degraded local limiter<br/>conservative quota = global_limit / N_instances<br/>serves traffic when Redis unreachable]

    EDGE --> LB
    LB --> L1
    LB --> SA
    LB --> SB
    SA --> LIB
    SB --> LIB
    L1 --> CORE
    LIB --> CORE
    CORE -->|EVAL atomic refill + check + decrement| RD
    RD --> POL
    CORE -->|fail-open / fail-closed decision, per route| DEG
```

Interviewer: Explain the three tiers and why you ordered them that way.

Candidate: Tier one is the local in-process limiter, and it is the reason this design scales. Instead of asking Redis on every request, each instance leases a block of quota from Redis. It asks for 1,000 tokens, holds them locally, and spends them one at a time as requests arrive. When it drops below a low-water mark it leases another block. The lease cost is amortised: 300,000 requests per second divided by 1,000 tokens per lease is 300 lease operations per second across the whole platform, which is nothing.

Candidate: The accuracy cost is a burst allowance equal to the lease size, so I size the lease as the per-second limit times a burst factor. For a 100 per second limit with a 300 burst, a lease of about 300 to 1,000 tokens is right, and I record that the true worst case is that an instance can be ahead of the global rate by up to one lease.

Candidate: Tier two is Redis as the authoritative global counter, consulted on lease refill rather than per request. Tier three is the degraded local mode, which I will explain when we discuss failure.

Interviewer: Where exactly does the limiter sit relative to the rest of the request handling?

Candidate: Order matters. Connection and TLS, then edge DDoS and WAF, then load balancer and its connection cap, then authentication, because I do not want to spend Redis calls on unauthenticated garbage, then rate limiting, then business logic, then the database. Rate limiting must be before any expensive work. I also put the platform-wide safety ceiling as the very last check just before the database, because that ceiling protects the database, not the customer.

Interviewer: And the API surface?

Candidate: It is mostly a library call, not an HTTP API, so the contract is a function:

```
Allow(ctx, req) Decision

Request:
  principal:
    userId    string   // may be empty
    tenantId  string   // may be empty
    apiKeyId  string   // may be empty
    clientIp  string   // IPv6-normalised
    tier      string   // free | premium | enterprise
  route:
    path      string
    method    string
  cost:
    tokens    int      // default 1, 10 for search, 100 for export

Decision:
  allowed        bool
  limits[]       { scope, limit, remaining, resetAt }
  retryAfterSec  int      // populated when allowed == false
  mode           enum    // LEASED | DIRECT | DEGRADED | FAIL_OPEN
  reason         string   // for logs, not returned to client
```

Candidate: And the client-facing response when it rejects:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 1
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1767525600
Content-Type: application/json

{ "error": "rate_limit_exceeded", "scope": "user", "retryAfter": 1 }
```

Interviewer: Why JSON at all for a rate limit response?

Candidate: For humans debugging in a browser, yes. I would keep it small and stable, and I would not leak internal policy details like which tier the user is on or the raw internal state.

## Phase 4 — Deep Dive

Interviewer: Algorithms. Walk me through the options and defend your choice.

Candidate: Five options, then my choice.

Candidate: Fixed window counter. Key is the current window index, value is a counter, TTL is two windows. One INCR, O(1) memory, the cheapest possible. Failure: allows 2x burst at the boundary, and a client can pick the boundary. Also with 300,000 requests per second you get a thundering herd on the key rollover.

Candidate: Sliding window log. Store an exact timestamped entry per request, O(limit) memory per key. Exact, no boundary burst. Failure: the memory math I did earlier, on the order of 43 gigabytes per hour of traffic, plus O(log n) trim operations. Reject.

Candidate: Sliding window counter. Two buckets, current and previous, and the effective count is the current plus the previous weighted by the fraction of the previous window still elapsed. O(1) memory, roughly accurate, and the boundary burst is bounded to 1x plus a fraction rather than 2x. Better than fixed window, still more bookkeeping than a token bucket.

Candidate: Token bucket. O(1) state, tokens refill at a constant rate up to a bucket capacity, each request costs some number of tokens. Burst is naturally bounded by the capacity. It is the only one of the five that expresses "cost weighting" as a first-class concept, which we need because search costs 10 and export costs 100. That single requirement is decisive for me.

Candidate: Leaky bucket, or its constant-rate variant GCRA. Tokens never accumulate, so the output is perfectly smooth at exactly the configured rate with no burst at all. Best for shaping traffic to a third-party API that must be protected from bursts. Failure: it forbids legitimate bursts, and a user who runs 100 requests once a minute gets throttled under a 100-per-second leaky bucket, which is bad product behaviour.

Interviewer: Why is that last point important? It sounds minor.

Candidate: It is the difference between a limiter users love and a limiter users route around. A token bucket with a capacity of 300 lets someone burst 300, recover over 3 seconds, and burst again, which matches how real traffic arrives. A leaky bucket refuses that entirely. So token bucket is the default, and I would offer leaky bucket or GCRA as a per-route policy for protecting fragile downstream dependencies. Study separately: [[rate-limiter|Rate Limiter]].

Interviewer: Now the atomicity. Why can't I just do a GET and then a DECR?

Candidate: Because between the GET that reads the current token count and the DECR, another request can interleave, and you will over-admit. With 300,000 requests per second arriving at 24 shards, that is thousands of concurrent interleavings per shard per second. Two round trips is a race, always.

Interviewer: So how do you make it atomic?

Candidate: A server-side script. One round trip, and the refill, the check, the decrement, and the TTL refresh all execute atomically inside Redis. I can do it with a Lua script via EVAL, which is the portable answer, or with a Redis Function, which is faster because it avoids shipping the script body on every call and is cached in the script cache. If Redis ships the GCRA module, RATE-LIMIT with Redis Cell does this in a single O(1) operation with about 24 bytes per key and no client-side arithmetic at all. Study separately: [[distributed-locks|Distributed Locks]].

Candidate: Here is the shape of the token bucket script.

```
KEYS[1] = rl:{tier}:user:{userId}      -- hash tag keeps it in one slot
ARGV[1] = cost tokens
ARGV[2] = refill rate, tokens per millisecond
ARGV[3] = bucket capacity
ARGV[4] = now_ms                        -- from Redis TIME, not the client
ARGV[5] = window_ms

-- inside Redis, atomically:
  now            = TIME based, never the caller's clock
  state          = HMGET key tokens last_refill
  elapsed        = now - last_refill
  tokens         = min(capacity, tokens + elapsed * rate)
  allowed        = tokens >= cost
  if allowed then tokens = tokens - cost end
  HMSET key tokens last_refill
  PEXPIRE key window_ms
  return { allowed, tokens, now + (capacity - tokens)/rate }
```

Candidate: Four things I want you to notice. The `min(capacity, ...)` clamp means idle users cannot bank unlimited burst. The TTL is refreshed on every call, so an abandoned key self-cleans. The reset time returned is computed from the current token deficit, not from a fixed boundary, which is what makes it a token bucket rather than a fixed window. And the hash tag in the key name is what makes it work on a Redis Cluster, where a script can only touch keys in one slot.

Interviewer: Tell me about that hash tag, because I have seen people get this wrong.

Candidate: In a Redis Cluster the key is distributed by the part inside the curly braces. So `rl:premium:user:12345` hashes the whole string, and a script that touches both `rl:premium:user:12345` and `rl:premium:ip:9.9.9.9` would be a cross-slot error and would be rejected. Putting the tag around the variable part, `rl:{premium:user:12345}`, forces both related keys into the same slot if I ever need a multi-key script. The design rule is: every key a script touches must share a hash tag. That also means I must not put the tier in the tag if I want tier changes to be a different key, so I have to decide deliberately. I choose to put scope and id in the tag and keep the tier as a prefix, so a tier upgrade changes the key but the old key expires on its own.

Interviewer: Clocks. You said use Redis TIME. Why not the local clock?

Candidate: Two reasons. First, correctness: if instance A's clock is 2 seconds ahead of instance B's, then a leased bucket refilled by A thinks it has more tokens available than it does, and the limit is violated. Every time-based decision must come from one clock. Redis TIME inside the script gives me that, and it means the client cannot be spoofed into claiming a stale time.

Candidate: Second, wall-clock pathologies. If NTP steps the clock backwards, elapsed goes negative, and depending on my implementation either tokens drain or the bucket gets stuck full. I clamp elapsed to zero on the negative side. I also clamp on the positive side, because a large forward jump would otherwise mint a full bucket of burst for free. Redis also supports relative expiry, and for local-only buckets I use a monotonic clock on the process so my own process never sees a step at all. Study separately: [[clocks-and-ordering|Clocks and Ordering]].

Interviewer: How do you handle the multiple dimensions, user and IP and tenant, without making four round trips?

Candidate: The leased model solves this too. Each instance leases quota for all the dimensions it needs in one lease, so four buckets is one lease operation, amortised. And for the multi-key concern, they may live in different slots, so I do not put them in one script; I do separate leases in parallel and combine with logical AND.

Candidate: I also want to be precise about which dimensions actually need to be globally accurate. User and tenant quotas are the commercial and abuse-control ones, so they are authoritative in Redis. IP limits are for volumetric bot control, and at that scale one IP is one key, so an approximate aggregate is fine. And the platform-wide ceiling is best enforced at the load balancer and at the database connection pool, not in this service at all, because a per-request check in the wrong place is the wrong place.

Interviewer: How do you size the lease and not break the limit?

Candidate: Lease size equals the limit times a burst factor, and the burst factor equals the natural burst I want to permit, which is exactly what the token bucket capacity is. For 100 per second with a 300 burst, I lease 300 tokens. The instance then spends 300 in under a second and has to lease again. The worst-case global over-admission is that each of the 300 instances can be up to one lease ahead, so 300 times 300 in a burst window. I bound it by setting the lease to a fraction of the capacity, say capacity divided by four, and by pacing refills to at most one per second per instance. The trade-off is a Redis call every 75 milliseconds per instance at full rate, which at 300 instances is 4,000 calls per second. Still nothing.

Interviewer: Let me push you on the exact numbers. Is that really 4,000 per second?

Candidate: Let me be careful. If an instance serves 300,000 divided by 300, which is 1,000 requests per second, and it leases 75 tokens per refill, that is about 13 refills per second per instance, times 300 instances, is about 4,000 lease operations per second. Yes. And if instead the lease is the full 300, it is about 3.3 refills per second per instance, so 1,000 per second platform-wide. The lease size directly trades Redis load against burst looseness, and the whole point is that the cost axis is 100x cheaper than per-request, so I can afford generous leases.

## Phase 5 — Trade-offs and Failure Scenarios

Interviewer: The Redis cluster dies. Do you fail open or fail closed? This is the question I actually care about.

Candidate: Neither, globally. Failing open and failing closed are both wrong as a single global policy, and the right answer is per-route, decided by the platform team as a policy, not by me as an implementation detail.

Candidate: The reasoning is that the cost of over-admission is not uniform. For login, signup, password reset, MFA verification, and coupon redemption, an abusive client costs real money or is a security incident, so fail closed: return 503 with Retry-After and let the client back off. Blocking legitimate logins for two minutes during a Redis blip is far cheaper than allowing credential stuffing.

Candidate: For the read APIs, most of the platform, and for health checks and internal service-to-service traffic, fail open with a local limiter. A few minutes of slightly higher traffic is not a user-visible incident, but a platform-wide 429 storm is. So fail open there.

Candidate: The middle option, and the one I would actually ship, is degraded mode rather than pure open or pure closed. When the circuit breaker on Redis opens, each instance computes a conservative local quota, global limit divided by the number of expected instances, times a safety factor of maybe 0.5, and enforces that locally. That means each instance allows 100 divided by 300, so under a third of a request per second, times 0.5, which is a very tight but non-zero budget. Legitimate single users doing 5 requests per second across 300 instances are not affected at all, because the local quota is not per user but a platform-wide drain rate on that instance. A single abusive client is still stopped, because their traffic concentrates on a few instances. This is strictly better than fail-open and it costs me nothing but a precomputed constant.

Interviewer: That is a good answer, but a client with 5 requests per second is now being throttled by an unrelated Redis outage. What is their experience?

Candidate: Their experience depends on whether the limit is enforced per instance or platform-wide. If I make the degraded quota platform-wide in intent but implemented per instance, then a user whose requests are spread across 300 instances is essentially unaffected, and a user concentrated on one instance gets the local share. So the degradation is proportional to how concentrated the abusive client is, which is the correct discrimination. I should also give the client a distinguishable error so we can alert, and I should not silently pretend the limit was precise.

Interviewer: You have a network partition, not a Redis death. Some instances can reach Redis and some cannot. What breaks?

Candidate: This is the nastier case, because I now have a split brain. Instances that can reach Redis see the true global count and enforce it strictly. Instances that cannot fall back to the conservative local quota. The result is that the true limit is enforced by some fraction of traffic and under-enforced by the rest, bounded by the local quota, so the platform can over-admit by at most the number of partitioned instances times the local quota. That is a real, quantifiable, bounded failure, which is much better than unbounded, and I would rather have bounded. I would also alert on partition specifically, because it is a different incident from a total outage and a different fix. Study separately: [[network-partition|Network Partition]] and [[split-brain|Split Brain]].

Interviewer: One IP is spraying 300,000 requests per second. Your per-key state is one hot key. What happens?

Candidate: If every one of those requests needs a lease, one key becomes 300,000 operations per second, which exceeds a single shard. Two mitigations. First, the leased model already helps enormously, because the hot client is concentrated on a few instances, so 300,000 requests are serviced by, say, 20 instances doing 15,000 each, and the lease rate is bounded by instance count, not by request rate. The lease model is inherently resistant to hot keys because the cost per key is per instance, not per request.

Candidate: Second, if I ever need finer granularity, I shard the key itself by adding a replica index, `rl:{ip:9.9.9.9}:3`, and pick a random replica per request, enforcing the limit divided by the number of replicas. The counts per replica are approximate, but the purpose here is stopping a flood, not metering it, and a random spread still blocks the flood within a factor of the replica count. I would also add an edge rule: any single IP above some absurd threshold, say 1,000 per second, gets blocked at the WAF and never reaches the app. Study separately: [[hotspot-handling|Hotspot Handling]].

Interviewer: A config error ships. Someone sets the free tier limit to 10 per second. What is your blast radius?

Candidate: The whole free tier throttles itself and support tickets arrive within minutes. Mitigations in order of speed. The policy is served from a pushed config with a version, so the fix is a config change, not a deploy. I keep the previous N good versions so I can roll back in seconds. I cap the change, so a single push cannot move a limit by more than a factor of 2, and anything larger requires two approvals. And I add a synthetic canary, a synthetic probe that makes real API calls as a free-tier principal, with an alert if it gets throttled, which catches the error in under a minute without waiting for users. And finally I cap the global damage, since a per-tier cap means a bad free-tier config cannot throttle premium or enterprise. Study separately: [[feature-flags|Feature Flags]].

Interviewer: Retry storm. A downstream service slows down, clients retry, and the retries amplify load. Does your limiter help or make it worse?

Candidate: It helps, and this is the strongest argument for limiting per endpoint and per dependency rather than only per customer. I can give the database-facing route its own much tighter bucket so that the platform cannot exceed the database's capacity no matter what clients do. That is a load-shedding valve pointed at the resource, and it works even if every client is misbehaving. I would also add an exponential backoff with jitter in the client library and require idempotency keys on retried writes, because a limiter that does not stop retries will still amplify them. Study separately: [[load-shedding|Load Shedding]] and [[idempotent-retry|Idempotent Retry]].

Interviewer: Why not a distributed lock instead of a Lua script?

Candidate: Because a lock is the wrong primitive. A lock means acquire, read, write, release, which is four round trips, four chances to fail, and a lock that leaks on a crashed client. A Lua script is one atomic operation with no lock to leak. I would also note that a naive SETNX-based lock with a short TTL is a well-known source of both races and availability problems, so if someone reaches for a lock here, the script is strictly better. Study separately: [[distributed-locks|Distributed Locks]].

Interviewer: What is the weakest part of your design?

Candidate: The degraded mode constant, and the fact that it is a guess. I hand-waved "divide by the number of instances times a safety factor". In reality the number of instances varies with autoscaling, the distribution of traffic across instances is not uniform, and the right constant is empirical. Worse, in degraded mode a legitimate low-rate user can be throttled, and that is a real product bug that I would have to accept consciously.

Candidate: The second weakness is the leased model versus a customer with a contractual guarantee. If a large enterprise customer's traffic is bursty, a lease of capacity divided by four means their peak is smoothed by my mechanism, not by theirs, and they may notice. The honest fix is a per-tenant direct mode with no leasing for the top few hundred tenants, which is cheap because it is few keys.

Interviewer: How do you know the limiter is healthy, and what does on-call see?

Candidate: Four signals, and the first one is unusual. Latency: decision latency p50, p99, p999, broken out by mode, because a p99 spike in DIRECT mode with a flat LEASED p99 tells me Redis got slow. Traffic: decisions per second by mode, and the ratio of LEASED to DIRECT, which is my early warning that Redis is struggling before requests are affected. Errors: circuit breaker state per shard, timeouts, and the count of degraded-mode decisions, which should be zero and any non-zero value pages someone. Saturation: Redis CPU and memory per shard, lease refill latency, and the count of instances holding a lease.

Candidate: Plus domain metrics that matter more: the number of distinct principals rejected per minute, which is your abuse signal and also your product signal, rejection rate by tier so a config mistake is visible, rejected-by-IP counts, and the count of calls to the hot-path overhead hook, because if that number ever goes up, something bypassed the limiter entirely, which is the failure I care about most. Study separately: [[golden-signals|Golden Signals]] and [[sli-slo-sla|SLI / SLO / SLA]].

Interviewer: We are at time. Summarise the design.

Candidate: The design is a three-tier limiter: a local in-process limiter that leases blocks of quota from Redis so that 300,000 requests per second costs only about 1,000 to 4,000 lease operations per second, a Redis cluster of 24 shards holding authoritative token buckets keyed with hash tags, and a degraded local mode that enforces a conservative per-instance quota when the backend is unreachable. The algorithm is a token bucket with weighted costs, chosen over fixed window, sliding window log, sliding window counter, and leaky bucket because it gives O(1) state, bounded burst, and it is the only one that expresses endpoint cost naturally, and it sits at the platform edge with a 5-millisecond hard timeout, a circuit breaker, and a bulkhead so that it can never be the cause of an outage. Correctness comes from a single atomic server-side script using one clock, and the failure policy is per-route, failing closed on authentication and payment routes, failing open on reads, and using degraded local limits in between.

## Study Separately

Interviewer: Which concepts do you need to read properly before this goes to a real panel?

Candidate: Study separately: [[distributed-rate-limiter|Distributed Rate Limiter]] — the central problem of correctness across instances, and the leased-quota variant in particular.

Candidate: Study separately: [[clocks-and-ordering|Clocks and Ordering]] — monotonic versus wall clock, NTP steps, and why every time-based decision needs one authority.

Candidate: Study separately: [[redis|Redis]] — Cluster slotting, hash tags, Lua versus Functions, the RATE-LIMIT cell module, and replication behaviour for a stateful counter.

Candidate: Study separately: [[bulkhead|Bulkhead]] — isolating limiter thread pools and connections so a slow backend cannot exhaust the caller's resources.

Candidate: Study separately: [[overload-protection|Overload Protection]] — where the platform-wide ceiling actually belongs, at the load balancer and the connection pool, not in this service.

Candidate: Study separately: [[sli-slo-sla|SLI / SLO / SLA]] — how to state "low single-digit milliseconds decision latency" as a measurable SLO with an error budget, rather than as an aspiration.

Candidate: Study separately: [[retry-and-timeout|Retry and Timeout]] — the interaction between timeouts, retries, and the 5-millisecond hard limit, since a retry inside the limiter is what turns a slow backend into an outage.

---

## What I Must Know

### Must Know
- [[rate-limiter|Rate Limiter]]
- [[redis|Redis]]
- [[distributed-rate-limiter|Distributed Rate Limiter]]
- [[overload-protection|Overload Protection]]

### Good to Understand
- [[clocks-and-ordering|Clocks and Ordering]]
- [[distributed-locks|Distributed Locks]]
- [[circuit-breaker|Circuit Breaker]]
- [[load-shedding|Load Shedding]]
