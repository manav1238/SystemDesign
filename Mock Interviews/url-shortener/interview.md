---
title: "Design a URL Shortener — Mock Interview Transcript"
status: active
tags: [hld, mock, url-shortener]
---

# Design a URL Shortener — Mock Interview Transcript

Read-only study transcript. Every line is a speaker turn. Reproduce the reasoning, not the words.

## Phase 1 — Requirements Clarification

Interviewer: Thanks for joining. Today we design a URL shortener. Users submit a long URL, get a short one back, and anyone who visits the short URL is redirected. Owners can set a custom alias, set an expiry, and see how many clicks their link got. We already have scale numbers: 100 million URLs stored, about 1 million new URLs a day, and roughly 10 resolutions per stored URL per day, so reads are about 10 times writes. Start anywhere you like, but take the first few minutes to ask me what you need to know.

Candidate: Thank you. Before I estimate anything, I want to understand the product, because two of these questions change the data model and I do not want to redesign later.

Interviewer: Go ahead.

Candidate: First, how long is the long URL on average? Real URLs with campaign tracking parameters are much longer than the 20 bytes people assume, and this is the dominant term in my storage estimate.

Interviewer: Say an average of 500 bytes, and up to 2 kilobytes in the tail.

Candidate: Good, that changes my storage math materially. Second, are custom aliases unique globally, or only within one user's namespace?

Interviewer: Let's say globally unique, but only for authenticated users. Anonymous users get random codes.

Candidate: That is important, because it means I have two different uniqueness domains and global alias uniqueness is an extra constraint I have to enforce. Third: can a link's destination be edited after creation?

Interviewer: Not in the first version. Delete and expire, yes. Edit, no.

Candidate: That simplifies consistency, because a cached redirect can never be wrong. Fourth: how fresh does the click count need to be?

Interviewer: Within a few seconds for a dashboard. It does not need to be exact to the click.

Candidate: Then the redirect path never blocks on analytics, which is the main thing I wanted to confirm. Fifth: is the read traffic global?

Interviewer: Yes, worldwide. A link shared in a Slack channel in Singapore has to resolve from Singapore.

Candidate: Then the hot path is dominated by network distance to my cache, and the cache has to be close to the user or globally replicated. Sixth, do links need to be revocable with a hard guarantee that the short code stops resolving, or is best-effort expiry acceptable?

Interviewer: Hard guarantee. If a campaign link has an expiry, it must stop working at that time.

Candidate: That rules out "we will sweep it eventually" as the only mechanism. Expiry has to be enforced on the read path, with a sweeper as a cleanup optimisation. Seventh, do we serve the short URL on its own domain or on a path of an existing domain?

Interviewer: Its own domain.

Candidate: Then I control DNS, the certificate, and I can serve a clean 302 with aggressive edge configuration. Eighth, how many application instances and how much of the traffic is anonymous?

Interviewer: Anonymous reads are the overwhelming majority. Creates are a tiny fraction.

Candidate: Then the create path can degrade much more than the redirect path, and I should build them as separately scalable services even if they share code.

Interviewer: That is a reasonable read. Anything else?

Candidate: Three more. Do we need to screen created URLs for malware and phishing, synchronously on the create path?

Interviewer: Asynchronously, but created links are untrusted for the first few minutes.

Candidate: Noted, I will design a quarantine state. Do we have a hard availability target for redirects?

Interviewer: 99.99 percent, single region, multi-AZ. And a p99 budget of roughly 100 milliseconds end to end.

Candidate: Understood. And last one: is it acceptable for the click pipeline to lose a small number of events if the broker is unavailable?

Interviewer: Yes, analytics is best-effort. Redirects are not.

Candidate: That gives me a clean priority order: redirect availability first, redirect latency second, create availability third, analytics accuracy last. Let me now restate requirements and do the estimation.

## Phase 2 — Estimation

Candidate: Let me state the functional requirements first so we are working from the same list.

Interviewer: Go ahead.

Candidate: Functional: create a short code for a long URL, resolve a short code to a redirect, optionally request a custom alias, optionally set an expiry, delete or deactivate a link, and view per-day click counts. Non-functional: redirects available 99.99 percent with p99 under 100 milliseconds, reads about 10 times writes, one code resolving to at most one destination with no edit path, expiry enforced as a hard guarantee, analytics eventually consistent within seconds and allowed to be lossy, and the system must scale horizontally with no shared mutable state in the redirect path.

Interviewer: Now the numbers. Show your arithmetic.

Candidate: Writes first. One million new URLs per day is one million over 86,400 seconds, which is about 11.6 per second on average. Peak is typically 3 times average for a consumer product, so I will design for roughly 35 write requests per second, and I will round to 50 to have headroom.

Interviewer: Reads.

Candidate: One hundred million stored URLs at 10 resolutions each per day is one billion resolutions per day. One billion over 86,400 is about 11,600 per second average. But redirects are not uniform, they follow the news cycle, so peak is more like 10 times average. That gives roughly 120,000 requests per second at peak. I will design for 150,000 to be safe.

Interviewer: That is a big number. What is the breakdown by path?

Candidate: Resolution is essentially all of it. Creates at 50 per second are 0.03 percent of peak traffic. So this is a read-mostly system where the write path is almost irrelevant to scaling, and the entire design effort belongs on the read path.

Interviewer: Storage.

Candidate: Let me be careful with the row size. A row has: short code 7 bytes, long URL 500 bytes average, owner id 8 bytes, created and expires timestamps 16 bytes, status and version about 40 bytes. That is roughly 570 bytes of payload. Add a secondary index on owner and creation time, about 50 bytes per row, and InnoDB page, MVCC, and row overhead at roughly 1.4 times. Call it 870 bytes per URL.

Candidate: One hundred million URLs at 870 bytes is 87 gigabytes. Round to about 90 gigabytes of primary. With three copies for replication, about 270 gigabytes, and with growth headroom for a few years I will plan for well under one terabyte.

Interviewer: So does that fit on one machine?

Candidate: It does, easily. A single modern database instance has multiple terabytes. So the honest conclusion is that sharding is not required at one hundred million URLs for storage reasons. But I am going to shard anyway for a different reason, which I will explain when we get there, and I want to be explicit that this is a judgement call, not a storage necessity.

Interviewer: Analytics volume.

Candidate: One billion clicks a day. A raw event with timestamp, code, referrer, country, user agent, and IP is around 200 bytes. That is 200 gigabytes a day of raw events, about 73 terabytes a year. I cannot store that indefinitely at that resolution.

Candidate: So I aggregate on the way in. Per code per hour, I estimated roughly 3 million distinct code-hour pairs a day, which is about 300 megabytes a day and 110 gigabytes a year. That is affordable in a columnar store. I keep minute resolution for seven days, hour resolution for thirteen months, and beyond that I keep only daily rollups. Hot keys get finer resolution because they are few and high value.

Interviewer: Bandwidth.

Candidate: The redirect response itself is tiny, maybe 300 bytes with headers. At 120,000 per second that is about 36 megabytes per second, roughly 290 megabits per second, about 3 terabytes a day. If I serve it from edge cache, the origin sees a small fraction of that. Not a bandwidth problem, but the request count is a connection and CPU problem, which is where connection reuse matters.

## Phase 3 — High-Level Architecture

Candidate: Here is the architecture. The single most important structural decision is that I split the system into a control plane and a data plane. The create path is the control plane and can be slow, can queue, and can depend on a database. The redirect path is the data plane and must be fast, stateless, and must never depend on a synchronous write.

```mermaid
flowchart TD
    DNS[Global DNS / Anycast]
    ELB[Edge LB multi PoP<br/>TLS termination, HTTP/3]
    RS[Redirect Svc<br/>data plane, stateless]
    CA[Create API<br/>control plane, auth, abuse]
    L1[L1 local cache<br/>hot keys only]
    KA[Key allocator / key pool manager]
    RD[Redis Cluster cache layer<br/>key: short_code -> {long_url, expires_at}, TTL=expiry<br/>per-shard Bloom filter of valid codes<br/>per-shard top-K hot codes in local memory]
    ST[Sharded URL store MySQL / Aurora<br/>PK short_code, index owner_id created_at<br/>primary + 2 read replicas per shard, multi-AZ<br/>soft delete / expire, never hard delete on path]
    ING[Ingest tier Kafka -> Flink<br/>-> ClickHouse / HBase keyed by code-hour<br/>-> daily rollups, top-K, public stats endpoint]
    SAF[Safety / threat-intel<br/>unfurl, malware scan<br/>link starts QUARANTINED -> ACTIVE]

    DNS --> ELB
    ELB --> RS
    ELB --> CA
    RS --> L1
    CA --> KA
    L1 --> RD
    KA --> RD
    RD -->|cache miss, bloom-negative short-circuit| ST
    ST -->|async, best effort, lossy| ING
    ST --> SAF
```

Interviewer: Walk me through the read path for a resolution, step by step, including what happens on a miss.

Candidate: Step one, DNS resolves the short domain, probably to anycast so the nearest point of presence answers. Step two, the edge load balancer or CDN checks its own cache. I am deliberately not caching redirects at the CDN for the default case, and I will come back to why, because the 301 and 302 decision drives it. For a 302, the CDN can cache with a short TTL, but the default configuration is no caching at the edge for correctness and for click accuracy.

Candidate: Step three, the request lands on a stateless redirect service instance. Step four, the instance checks its L1 in-process cache, which only holds a curated set of hot codes plus codes this instance has seen recently. Step five, on an L1 miss it does a single Redis GET. Step six, if Redis misses, it consults the shard's Bloom filter. If the Bloom filter says the code is absent, the answer is 404 with no database access at all. If the Bloom filter says maybe, we go to the database.

Candidate: Step seven, on a database hit we populate Redis with a TTL, populate L1, and return. Step eight, the click is counted asynchronously, and only after the response is on the wire. The redirect path never writes to the database and never waits for the analytics system.

Interviewer: Why not count the click synchronously with an INCR? It is one call and you said reads are only 10 times writes.

Candidate: Two reasons. First, correctness of the user-visible latency budget: adding a synchronous Redis write to the hot path doubles the number of network round trips on the hottest path in the system. Second, failure coupling: if the click counter is slow or unavailable, and I have put it on the synchronous path, redirects slow down or fail. Redirects must not depend on the analytics path at all. I will put the increment on a local in-memory ring buffer flushed by a background thread, so the request path pays a memory write and nothing else.

Interviewer: The buffer is lost if the instance dies. How many clicks do you lose?

Candidate: Bounded by the flush interval. I flush every 200 milliseconds, so I lose at most 200 milliseconds of counts for that instance, and I accept it. For durable counting I would add a per-key counter in Redis incremented in the background flusher, so the loss window is 200 milliseconds of unflushed buffer plus nothing else, because the flush itself is acknowledged.

Interviewer: Now the create path.

Candidate: Create is authenticated, it is slow-path, and it has four stages. Stage one, validate the URL: scheme must be http or https, block known-bad destinations, normalise the URL by lowercasing the host, stripping the default port, and removing common tracking parameters if the product wants that. Stage two, acquire a code, either from the key pool for a random code or by validating a custom alias against the alias registry. Stage three, write to the primary database with a synchronous replica acknowledgement, and write through to the cache so that the very first read after creation is a hit. Stage four, return the short URL.

Candidate: The write-through to cache on create is a detail I want to defend. If I only invalidate on create, the first resolver after creation is a cache miss, and if that read goes to a lagging replica, the brand-new link 404s. Write-through on create makes reads-after-write correct for the common case without requiring every read to hit the primary.

## Phase 4 — Deep Dive

Interviewer: Let's go deep. Start with how you generate the short code, and show me the collision math.

Candidate: The requirement is a short, URL-safe, hard-to-guess-if-I-want-it-hard-to-guess code. The alphabet is base62, which is 0-9, a-z, A-Z, 62 characters. With 7 characters I get 62 to the 7th power.

Interviewer: That's about 3.5 trillion.

Candidate: About 3.5 trillion, yes. Here is the trap, and I want to be explicit about it because this is where most candidates get it wrong. You cannot use all of them. With 100 million codes in a space of 3.5 trillion, the birthday bound says expected collisions are roughly n squared over 2m, which is 1e16 over 7e12, that is about 1,400 collisions. Generating random 7-character codes and retrying on collision is fine at 1,400 collisions total across the lifetime, but the retry path means a random insert race, and at higher scale it degrades badly. Let me show the right numbers.

Candidate: For negligible collision probability at 100 million keys I need roughly n squared over 2m much less than 0.01, which pushes me to about 9 characters, since 62 to the 9th is 1.35e16. Nine characters is a long short URL. So random generation with retries is the wrong default at this scale.

Candidate: My actual approach has two tiers. Tier one, the standard path, is a pre-allocated key pool, which is what the large operators do. A background job generates codes in bulk, checks them against the database once, and stores them as AVAILABLE. A create request leases a block of maybe 10,000 codes, hands them out in memory, and asynchronously returns them to a USED or RECYCLED state. Collision is impossible by construction because availability is verified before the key is ever handed to a user, so there is no concurrent random insert race. The cost is a background job and a pool table, and the benefit is that create is a memory pop.

Interviewer: And tier two.

Candidate: Tier two is the custom alias, which must be globally unique. That is a uniqueness constraint on user input, so it is an ordinary unique index plus a friendly error. The subtlety is the lock: I need to be careful with concurrent requests for the same alias, and the cleanest way is a unique constraint on short_code in the database, relying on the storage engine to reject the loser. I do not want a separate distributed lock for this. I would also rate-limit custom alias attempts per user to stop alias squatting by scanning.

Interviewer: Walk me through the full create API contract.

Candidate: Two endpoints. The public one:

```
POST /v1/urls
Headers:
  Idempotency-Key: <uuid v4>
  Authorization: Bearer <token>   (optional, enables custom alias + analytics)
  X-Request-Id: <uuid>            (optional, for tracing)
Body:
  {
    "longUrl": "https://example.com/very/long/path?utm_source=x&utm_medium=y",
    "customAlias": "q1-promo",          // optional, authenticated only
    "expiresAt": "2026-04-01T00:00:00Z", // optional, null means never
    "title": "Q1 campaign",             // optional, owner label
    "projectTag": "marketing"            // optional
  }
201 Created
  Location: /v1/urls/aB3xY9z
  {
    "shortCode": "aB3xY9z",
    "shortUrl": "https://sho.rt/aB3xY9z",
    "longUrl": "https://example.com/...",
    "createdAt": "2026-01-04T10:15:00Z",
    "expiresAt": null,
    "status": "ACTIVE"
  }
409 Conflict      alias already taken
422 Unprocessable  bad scheme, blocked destination, alias too long
429 Too Many Requests   per-user create limit
```

Candidate: The idempotency key matters here, because create is a network call that the client will retry on timeout, and without it a retry creates a second link and a second code. I store the idempotency key with the user and return the original response for a repeat. I made it required, not optional, because the cost of a duplicate is a permanently orphaned link.

Interviewer: And the read side.

Candidate: The resolve path is a GET on the code, but it is deliberately not a JSON API, it is a redirect:

```
GET /{shortCode}
302 Found
  Location: https://example.com/very/long/path
  Cache-Control: no-store
  X-RateLimit-Policy: none
```

Candidate: Plus management and analytics endpoints:

```
GET    /v1/urls/{code}                 -> owner-visible metadata
PATCH  /v1/urls/{code}                 -> change title/tag only, not destination
DELETE /v1/urls/{code}                 -> soft delete, sets status DELETED
GET    /v1/urls?limit=50&cursor=...    -> list owner's links
POST   /v1/urls/{code}/stats:query     -> per-day clicks, referrers, countries
```

Candidate: One design choice to call out: GET on the code is a redirect, not a JSON object. A JSON body plus a status code means the client has to do a second call to learn the destination, which doubles latency and engagement. A 302 puts the destination in the Location header, which is exactly what browsers, messengers, and Slack unfurlers consume natively.

Interviewer: Now the database schema.

Candidate: Here is the core table.

```
CREATE TABLE urls (
  short_code      CHAR(7)       NOT NULL,
  long_url        VARCHAR(2048) NOT NULL,
  canonical_hash  BINARY(32)    NOT NULL,      -- SHA-256 of normalised URL
  owner_id        BIGINT        NULL,          -- null for anonymous
  status          TINYINT       NOT NULL,      -- 0 QUARANTINED 1 ACTIVE 2 EXPIRED 3 DELETED
  created_at      DATETIME(3)   NOT NULL,
  expires_at      DATETIME(3)   NULL,
  deleted_at      DATETIME(3)   NULL,
  click_count_big BIGINT UNSIGNED NOT NULL DEFAULT 0,   -- denormalised rollup
  version         INT           NOT NULL DEFAULT 0,     -- optimistic concurrency
  PRIMARY KEY (short_code),
  KEY idx_owner_created (owner_id, created_at DESC),
  KEY idx_canonical (canonical_hash),
  KEY idx_expiry (expires_at)
) ENGINE=InnoDB;
```

Candidate: Decisions here. Primary key on short_code, clustered, so every read is a single B-tree seek, which is exactly the access pattern. owner_id is nullable and indexed for the list endpoint. I deliberately denormalise click_count_big onto this row even though analytics lives elsewhere, because the owner's link list needs to show counts, and joining the analytics store on that hot path would be slow. It is updated by the aggregator with a periodic flush, not per click, so it is explicitly eventually consistent. And idx_expiry exists to make the cleanup sweeper cheap.

Interviewer: Why is the click count on the row at all if you said analytics is separate?

Candidate: Because the owner's list view shows 20 links with their counts, and doing 20 cross-store lookups per page is a latency problem and a fan-out problem. The row value is a cache of a rollup, updated every minute or so. It is a denormalisation with a stated staleness contract, not a source of truth. I will be explicit in the doc that it is not accurate for billing-grade use.

Interviewer: Now caching in detail. Tell me about the cache hierarchy and the three-layer design.

Candidate: Three layers. Layer one is the L1 in-process cache inside each redirect instance, a small LRU of a few thousand entries holding only hot codes, with a short TTL of about 30 seconds. Layer two is the Redis cluster, the main cache, with roughly 32 shards, replicated, holding the full hot set. Layer three is the database.

Candidate: Why L1 at all if Redis is fast? Three reasons. First, it removes a network round trip from the hottest path, which is real latency and real connection pressure at 120,000 requests per second. Second, it lets one viral key be served by every instance from local memory, which is the cheapest possible hot-key defence. Third, it degrades gracefully, because if Redis has a bad minute, L1 keeps serving the top keys.

Interviewer: What is the cache fill policy, and what TTL do you use?

Candidate: Cache-aside. On a miss, read the database, then SET with a TTL. The TTL has two components. A freshness floor of 60 seconds, so that a destination change or a delete propagates, and a cap at the link's own expires_at, because a cached entry must never outlive the link's expiry or I break the hard expiry guarantee. So TTL equals the minimum of 60 seconds and the time remaining until expiry. For a link expiring in 20 seconds, TTL is 20 seconds. I also jitter the TTL by plus or minus 10 percent, so thousands of entries created together do not expire together.

Interviewer: Tell me about negative caching, because I think you are missing something.

Candidate: I was about to get to it, and you are right that I need it. A resolver that gets 404 should cache that 404 for a short period, say 15 seconds. Otherwise a scanner or a typo tool that generates random codes produces a full database read for every single request, and that is an easy denial of service on my read path. So negative caching is not a nice-to-have, it is a DoS control. And the Bloom filter is the structural version of the same idea.

Interviewer: Explain the Bloom filter part properly.

Candidate: Per shard, I keep a Bloom filter of valid codes in that shard. On a request, I check the filter before the database. If the filter says definitely-absent, I return 404 without touching the database at all. If it says maybe-present, I go to the database.

Interviewer: Sizing.

Candidate: I want about a 1 percent false positive rate, so roughly 10 bits per key. For one shard, one hundred million codes over, say, 64 shards is about 1.6 million codes per shard, so 1.6 million times 10 bits is 16 million bits, which is 2 megabytes per shard, 128 megabytes total. Trivial to hold fully in memory, and I can hold it on every redirect instance, so the check is a local memory lookup with zero network cost.

Candidate: The maintenance problem is the interesting part. A standard Bloom filter cannot delete, so when a link is deleted or expires I cannot remove it and I accumulate false positives over time. Two options. Option one, a counting Bloom filter, where each position stores a small counter incremented on add and decremented on delete, which is about 3 to 4 times the memory but supports deletion. I would use this. Option two, rebuild the filter from the database on a schedule, which is simple but the rebuild must be atomic, because serving from a half-loaded filter means returning false 404s to live links, which is the worst possible bug in this system.

Interviewer: That false-404 risk is worth dwelling on.

Candidate: Yes. A Bloom filter that is incomplete is more dangerous than no filter, because it converts a miss into a wrong answer rather than into a slower answer. My rules are: the filter is loaded before the service takes traffic, the load is a full scan of active codes per shard, and it is rebuilt in the background into a new generation, swapped atomically, and the old one is kept until the new one is fully loaded. I also add a safety valve: if the filter is not marked ready, the code path bypasses the filter and goes to the database, degrading to slow-but-correct.

Interviewer: Now partitioning and sharding. You said you would shard at 100 million URLs even though it fits. Explain.

Candidate: Correct, and the reason is not storage. The reason is the write path. I have a single logical table whose hot operation is a random point read, and if I keep it on one primary I have a single point of failure, a single ceiling on IOPS, and a replication cost that grows linearly. Sharding lets me scale reads linearly, lets me place each shard in a different availability zone, and lets me fail over one shard without taking the service down.

Candidate: The shard key is the obvious one, short_code, because that is the only access pattern that matters on the hot path. Short codes are high-entropy and roughly uniform, so hash sharding by short_code gives even distribution with no manual range management. Hash short_code, take the first two bytes of the digest, mod 64. That is a clean consistent-hash-free mapping, a simple modulo map, with a directory that maps shard number to host list.

Interviewer: Modulo sharding has a well-known problem.

Candidate: Yes, modulo resharding means when I add a shard from 64 to 128 I have to move every key, because the mapping changes for all of them. With 100 million keys that is a full rebalance. I would start at a fixed shard count, say 64, chosen with headroom for several years of growth, and keep the count stable. If I need to grow past 64 shards, I switch to consistent hashing with virtual nodes, which only moves about one in N keys. The honest answer is: pick a fixed count big enough that you can defer the rebalancing problem, and write the routing layer so that swapping modulo for consistent hashing later is a config change plus a data migration, not an architecture change.

Interviewer: What about the secondary access patterns? Owner listing and alias lookup do not have the shard key.

Candidate: Two problems, two solutions. Owner listing: the idx_owner_created index lives inside the shard that owns each row, so a query for "my links" is a scatter-gather across all 64 shards. That is acceptable because it is a low-QPS, owner-only, paginated read, and the common case is a user with tens of links, so 64 parallel small queries is fine. I cap page size and I have a timeout so it degrades rather than hangs.

Candidate: Alias lookup, which is really just a read of the same short_code, so it routes to the same shard. No problem.

Interviewer: The 301 versus 302 question. This matters and people get it wrong.

Candidate: 301 Moved Permanently tells the client to cache the redirect forever. Browsers obey this aggressively, and once cached the client never comes back to my server. That is the cheapest possible serving cost, and for a permanent, unexpiring, unchanging link it is genuinely the right answer.

Candidate: But it is wrong for the default case, for three reasons. First, I lose the click. If the browser caches the 301, I never see the request, so the owner's analytics undercount forever. That destroys the analytics product. Second, I lose control. If I later need to delete the link, block a malicious destination, or apply an expiry, the browser will not ask me, so my hard deletion and hard expiry guarantees are gone. Third, 307 and 308 are the method-preserving equivalents, and if a link is ever used in a POST flow, 301 can convert the method to GET.

Candidate: So my default is 302, always. I expose an opt-in flag, permanent, that returns 301 and also explicitly disables analytics and sets a very long edge TTL. And I add `Cache-Control: no-store` on the 302 so the client does not build a private cache entry on its own. Study separately: [[http-caching|HTTP Caching]].

Interviewer: Replication. How many copies, and sync or async?

Candidate: Three nodes per shard, primary plus two replicas, in three separate availability zones. Writes go to the primary and are acknowledged by at least one replica synchronously, so an acknowledged write survives losing an entire availability zone. That costs write latency, roughly one to two milliseconds of extra round trip, and I only care about writes on the create path at 50 per second, so I can afford strong durability there. Reads go to replicas, which is fine because the mapping is immutable after creation, so a stale replica is not a correctness problem, only a freshness problem for the metadata.

Interviewer: Careful. You said you write through to cache on create, so reads should not hit the database. But when they do hit the database, which node answers?

Candidate: Replica, and here is the subtlety worth stating: because the mapping never changes, the worst a replica can serve is a link that was created less than the replication lag ago, and replication lag is typically tens of milliseconds. But if a user creates a link and clicks it within 200 milliseconds, they are inside that window and they get a 404, which looks exactly like a broken product. My protections are three. One, write-through to cache on create, which covers the overwhelmingly common case. Two, route reads for codes created in the last few seconds to the primary, which I can do cheaply with a small in-process set of recently-created codes kept warm on the create path. Three, Bloom filters do not cause false negatives if they are correct, so a "maybe" always goes to the database and a 404 only happens if the row genuinely is not there. Study separately: [[replication-lag|Replication Lag]].

## Phase 5 — Trade-offs and Failure Scenarios

Interviewer: Traffic just doubled overnight. Where does it break first, and what do you do?

Candidate: Let me rank by how close each component is to its limit. The redirect instances are at 120,000 requests per second across, say, 40 instances, which is 3,000 per instance. A single instance comfortably handles 10,000 to 20,000 requests per second of this workload, so we are at maybe 20 percent utilisation. The Redis cluster at 32 shards is handling the cache-hit path, and I need to be honest: my 95 percent hit rate assumption is what keeps this cheap. If the hit rate falls to 80 percent, Redis requests per second goes from 6,000 to 24,000, and the database goes from 6,000 to 24,000 reads per second, which the shards can handle but at real cost.

Candidate: So the answer is: nothing breaks first if the cache hit rate holds, and the whole system's fragility is the cache hit rate. My first action is not to add redirect instances, it is to look at why the hit rate moved. The usual causes are a new traffic source with a very different key distribution, which pushes the working set out of memory, or a cache eviction policy that started throwing away hot keys.

Candidate: For the doubling itself, the levers in order are: raise the L1 hit rate, because it is free; add Redis shards, because it is cheap and rebalances automatically; add redirect instances behind the load balancer; and only then add database shards. Adding redirect instances is trivially reversible, so I scale that first to buy time.

Interviewer: The primary database for one shard dies. Walk me through the exact sequence.

Candidate: Sequence. The synchronous write path notices the connection drop. The engine promotes the replica with the highest applied transaction log, which in a single-AZ-per-replica setup is the replica in the second zone. Promotion takes tens of seconds. During that window, that shard cannot serve reads from the replica with current data, and it cannot accept writes.

Candidate: What users experience: roughly 1 in 64 of all codes, by hash, return 503 with a Retry-After header. Not a full outage, which is why sharding by short_code with 64 independent shards is a genuinely valuable property here. The Bloom filters and Redis cache for that shard still work, because the cache is a separate system with its own replication. So in practice, for the roughly 95 percent of traffic that is a cache hit, users see nothing at all during failover. Only cache misses on the failed shard see an error.

Candidate: The client-side behaviour I want is a short circuit, not a retry storm. A 503 with Retry-After of 2 seconds, plus a client that honours it, is much better than an immediate retry. And I add a circuit breaker on the database call so a failing shard does not consume the connection pool of the redirect instance. If I did not shard, this would be a 100 percent outage instead of a 1.6 percent one. Study separately: [[failover|Failover]] and [[circuit-breaker|Circuit Breaker]].

Interviewer: One link gets a million clicks in a minute. What actually happens, and is it a problem?

Candidate: Let me compute rather than panic. A million clicks in a minute is about 17,000 requests per second on one key. One Redis key at 17,000 operations per second is well within a single node's capability, which is typically 100,000 operations per second or more, so Redis does not actually fall over. The database, if the cache missed, would be asked for 17,000 point reads per second on one row. That is a problem, because a single B-tree page hot-spots, and the row lock and buffer pool contention on one page becomes a real bottleneck.

Candidate: So the real defence is making sure we never reach the database for that key. Three things. One, L1 caches on every instance absorb the traffic locally with no network hop at all, which is the biggest win and the cheapest. Two, I detect hot keys by tracking request rate per code in the aggregator, and when a code crosses a threshold I proactively pre-warm it into every instance's L1 and pin it in Redis, and optionally replicate the cache value to a second key with a different hash suffix so the traffic is split across two shards. Three, I put a specific rate limit on the resolve path per code, not just per user, so a single code cannot exceed some ceiling even if it is legitimately viral.

Interviewer: A cached entry for that viral key expires and five hundred thousand requests all miss at once. What is that?

Candidate: That is a cache stampede, and it is one of the classic failure modes. The standard answers, in the order I would implement them. First, jittered TTLs, which I already have, so entries do not expire simultaneously. Second, early recomputation, so the serving instance refreshes slightly before the TTL runs out rather than at the moment it runs out. Third, and this is the one that actually matters, request coalescing, which is a per-key mutex so that the first miss takes the lock and does the database read, and every other miss waits on that same in-flight read instead of issuing its own. A singleflight pattern.

Candidate: Fourth, stale-while-revalidate, which is serving a slightly stale value from L1 while one instance refreshes, trading a little freshness for total availability. And fifth, negative-result caching on errors, so if the database is struggling, the failures are cached for a few seconds rather than turning into a database stampede. Study separately: [[cache-warming|Cache Warming]] and [[hotspot-handling|Hotspot Handling]].

Interviewer: Why not use a distributed cache for everything and skip the database reads entirely? Why not a NoSQL store in place of MySQL?

Candidate: Because the database is the source of truth and the cache is a derived, disposable copy. The moment you treat the cache as the system of record, you have made cache loss a data loss event, and you have made cache inconsistency invisible until it becomes an outage. For the store itself, a key-value store like DynamoDB is genuinely a reasonable fit for the mapping, since the access pattern is a single-key point read, and I would not argue that MySQL is sacred. But a KV store gives you a unique index and a foreign-key-free table almost for free, and it gives you nothing for owner listing or expiry sweeps without extra indexes, and the cost model of provisioned throughput versus on-demand reads is a real budget conversation at 24,000 reads per second. I would start on MySQL because I can shard it in the ways I control, and revisit the KV store once the read volume forces the issue.

Interviewer: What is the weakest part of your design right now?

Candidate: The expiry cleanup sweeper, and I want to be honest about it rather than hide it. The hard guarantee is enforced on the read path, because every cache entry's TTL is capped at the link's expires_at, and a database read also checks expires_at and returns 410. That part is solid.

Candidate: The weak part is the background sweeper that flips status to EXPIRED and reclaims keys to the pool. If the sweeper is slow, deleted links linger in the database, which is a cost and a compliance issue, but not a correctness issue, because the read path is still correct. And the reclamation is genuinely hard, because the key pool must not hand out a code that is still resolvable from an old cache entry, or we resurrect a dead link. My rule is that a reclaimed key only goes back to the pool after its maximum possible cache TTL has fully elapsed, which I enforce with a delayed queue message scheduled at expiry plus 10 minutes. Study separately: [[idempotent-consumer|Idempotent Consumer]].

Interviewer: Last question. How do you know the system is healthy, and what does an on-call engineer see?

Candidate: Four golden signals per component. Latency: p50, p99, p999 for resolve, with resolve broken out by cache hit, L1 hit, and database hit, because a rising p99 with a stable hit rate is a different incident from a p99 caused by a hit-rate collapse. Traffic: requests per second and, more importantly, hit rate as a first-class metric, since I argued earlier that hit rate is the health of this system. Errors: 4xx by code, and 5xx by shard, so I can see "shard 47 is failing" rather than "1 percent of errors". Saturation: Redis memory and evictions per second per shard, database connections per shard, and Bloom filter readiness per instance.

Candidate: Plus three domain-specific alerts that matter more than the generic ones: a Bloom filter not ready on any instance, a sweeper lag over some threshold in minutes, and a replication lag spike on any primary, because that is the one that causes the fresh-link 404 I described. And a distributed trace on the resolve path with the three hop types as spans, so a slow request is immediately attributable to L1, Redis, or the database.

Interviewer: We are at time. Give me your final architecture in a couple of sentences.

Candidate: The design is a stateless, horizontally scaled redirect service sitting behind global DNS and an edge load balancer, backed by a three-tier read path of an in-process L1 cache for hot codes, a Redis cluster, and a sharded MySQL store that is the source of truth, with a Bloom filter in front of the database to make invalid-code lookups free. The create path is a separate control plane that hands out codes from a pre-generated key pool, writes synchronously with a replica acknowledgement, and quarantines new links for safety screening, while click analytics run entirely asynchronously through a Kafka ingest tier into an aggregated columnar store that the redirect path never waits on. Expiry is guaranteed on the read path by capping cache TTLs at expires_at, sharding by short_code means losing a primary is a 1-in-64 error rate rather than an outage, and the entire system's health reduces to one number: the cache hit rate.

## Study Separately

Interviewer: You used a lot of concepts quickly here. Which ones do you want to go read properly before the next interview?

Candidate: Study separately: [[consistent-hashing|Consistent Hashing]] — I dodged it with modulo and a fixed shard count, and I should be able to defend when that stops working.

Candidate: Study separately: [[probabilistic-data-structures|Probabilistic Data Structures]] — the counting Bloom filter and its deletion story, and what false negatives would cost me.

Candidate: Study separately: [[idempotency|Idempotency]] — the Idempotency-Key mechanics and how long I retain the key mapping.

Candidate: Study separately: [[hotspot-handling|Hotspot Handling]] — key replication, request coalescing, and pre-warming as techniques in their own right.

Candidate: Study separately: [[normalization-vs-denormalization|Normalization vs Denormalization]] — the denormalised click_count column and the exact staleness contract I promised.

Candidate: Study separately: [[cap-theorem|CAP Theorem]] — I chose to give up availability for the write path and eventual consistency for the read path, and I should be able to argue that as a deliberate PACELC trade rather than a slogan.

Candidate: Study separately: [[cdn|CDN]] — I explicitly chose not to edge-cache the 302 by default, and I need the reasoning for cache key design and TTL to be sharper.

---

## What I Must Know

### Must Know
- [[caching|Caching]]
- [[sharding|Sharding]] and [[shard-key|Shard Key]]
- [[consistent-hashing|Consistent Hashing]]
- [[redis|Redis]]

### Good to Understand
- [[probabilistic-data-structures|Probabilistic Data Structures]]
- [[database-replication|Database Replication]]
- [[http-caching|HTTP Caching]]
- [[capacity-estimation|Capacity Estimation]]
