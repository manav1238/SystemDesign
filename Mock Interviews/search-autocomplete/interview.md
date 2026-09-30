---
title: Search Autocomplete System Design - Interview Transcript
status: active
tags: [hld, mock, search-autocomplete]
---

# Search Autocomplete System Design - Interview Transcript

Format: full 45-minute mock interview. Every line is spoken dialogue.

---

## 0:00 - 0:04 - Opening and the Problem

Interviewer: Forty-five minutes, one problem. I'm going to describe a product and I want to hear your reasoning, not a memorized answer. Interrupt me if you want clarification, and I'll interrupt you.

Interviewer: Build the typeahead for a large consumer app. One search box, users type, a dropdown of five to ten suggestions appears and updates as they type. It has to feel instant, it has to show what people are actually searching for right now, and it should get more useful the more a particular person uses it. The vocabulary of distinct queries is about a billion, and it changes constantly because trends spike. Go.

Interviewer: Questions before you draw?

Candidate: I want to nail down the interaction contract first, because autocomplete is a fundamentally different problem from search and most people blur them. Three things: what is the keystroke-to-suggestion latency budget, what happens on an empty or one-character prefix, and is the interaction "type and press enter" or "arrow into a suggestion and keep typing", which means autocomplete versus query rewrite.

Interviewer: Budget is tight, call it 100 milliseconds p99 from keystroke to rendered dropdown, and the client debounces. Yes to arrowing in, so the returned string has to be a real completion that extends what the user typed, and we need the display metadata. On a zero or one character prefix, no suggestions, the client shows trending searches instead, that is a separate list. Trends can take up to five minutes to reflect, but a genuinely popular query appearing in under five minutes is a real complaint we track. Multiple locales and three product verticals, and the vocabulary overlaps between verticals but diverges. Safety filtering is mandatory and blocked terms must never appear.

Candidate: Good. Let me write my assumptions.

Candidate: One: suggestions come from actual search query logs, not from a curated dictionary. Two: eventual consistency is fine, five minute staleness is acceptable, and I will engineer for one minute. Three: the top-K contract is 10 results per prefix, K is a parameter. Four: I have access to a Redis or memcached-class tier, a wide-column or document store for larger structures, and a batch and stream processing layer. Five: multi-region with edge presence is in scope because this endpoint runs at enormous volume and 100 milliseconds is unforgiving of a cross-ocean round trip.

Interviewer: Fine. Scale.

---

## 0:04 - 0:13 - Phase 2: Estimation

Candidate: Let me build the request model bottom-up, because the keystroke multiplier is the number that surprises people.

Candidate: Assume 500 million daily active users on the search surface, and an average of 6 search sessions per user per day, so 3e9 sessions per day. Within a session, a user types a query averaging maybe 15 characters. Now, keypresses are not requests. The client debounces at roughly 150 to 200 milliseconds, and a typist at 200 milliseconds per character does not produce two overlapping requests, so with a 150 millisecond debounce I get roughly one request per character after the initial burst. So a 15-character query is about 14 suggestion requests, not 15 sessions, and one at the end.

Candidate: 3e9 sessions times 14 requests is 4.2e10 requests per day. Divide by 86,400, that's about 486,000 requests per second average. Peak is maybe 2.5x for evening usage, so design peak is about 1.2 million suggestion requests per second. That is a genuinely enormous number of requests, and it is the number that dictates that the serving path must be a memory lookup and nothing else.

Candidate: I should sanity check by bandwidth instead. A response with 10 suggestions, each roughly 60 bytes with an ID, text, and a category, plus JSON overhead, is about 800 bytes to 1 kilobyte. 1.2 million requests per second times 1 kilobyte is about 1.2 gigabytes per second, roughly 100 terabytes per day egress. That is fine over a CDN-style edge network, and it is another reason this must be served close to the user rather than from a single region.

Candidate: The vocabulary. A billion distinct queries, and it is heavily long-tailed. Let me assume the top 100 million queries account for 99 percent of traffic, and the remaining 900 million are the tail, which together contribute about 1 percent. That asymmetry is the entire design, because I am going to make a decision that the tail is not worth keeping in the hot serving structure.

Candidate: Now the memory estimate, which is the decision I most need to get right. I want a trie or a prefix tree over the top 100 million queries. A query averages 20 characters. A naive trie has one node per character per distinct prefix, and for a corpus like this the number of distinct prefixes is roughly 2x to 3x the number of distinct queries, so 200 to 300 million nodes. If each node is an object with a hash map of children, that is easily 100 to 200 bytes per node in a typical language runtime, which is 20 to 60 gigabytes just in object overhead before you store a single payload. That is too fat.

Candidate: Compact it and it gets reasonable. A flat array-based trie with children stored in a contiguous sorted block and child lookup by binary search, or a double-array trie, gets to roughly 16 to 32 bytes per node, which is 5 to 10 gigabytes for 300 million nodes. Add the payload: if I store a suggestion ID per node rather than the text, 4 bytes, that's 1.2 gigabytes more, and the text lives in a separate string table. So the hot structure is roughly 8 to 12 gigabytes of highly compact memory. That is a handful of machines, and now I can shard it.

Candidate: Let me also size the full vocabulary for the tier below. 1e9 queries, each with a count, a set of time-windowed counts, and a language and vertical tag. Say 50 bytes per query, 50 gigabytes. With three time windows and a couple of signals, call it 150 gigabytes. That is a disk-resident, wide-column, sharded store, and it is what you fall back to for the tail and what you use to rebuild the hot tier.

Candidate: And the precomputation volume. Every suggestion request is a prefix lookup, and a naive recompute of "all prefixes of all queries" is 1e9 queries times 20 prefixes equals 2e10 prefix-entries per rebuild. That is the precomputation cost, and it tells me I cannot recompute the full structure per user and I cannot recompute it per request. It has to be a batch job over logs, which is exactly the batch-versus-stream tradeoff.

Candidate: Read-write ratio here is essentially infinite. Zero user writes, all reads, plus a continuous stream of log events coming in. This is a read-only serving system with a write pipeline. That framing matters, because a read-only system has very different failure modes: nothing is lost when a serving node dies, we just have less capacity.

Interviewer: Non-functional requirements?

Candidate: Latency: 100 milliseconds p99 keystroke to paint, which after network means single-digit milliseconds server-side. The entire budget should be spent on one memory lookup, not on a fan-in. Availability: this is a conversion surface, so a failure here costs revenue directly, and I would run at 99.99 on suggestions with a graceful degradation to trending-only if the suggestion service is down rather than a spinner. Consistency: eventual, bounded to about one minute for the popularity signal. Correctness: this is a hard requirement, not a trade-off, a blocked term must never be returned under any failure, so filtering is baked into the precomputed output, not applied at request time. Durability: the query log is the durable truth, the trie is a rebuildable artifact. Privacy: personal signals are per user and never leave their scope, and a user's history-derived suggestion must not be visible to another user.

Study separately: [[read-write-ratio|Read/Write Ratio]], [[memory-estimation|Memory Estimation]], [[latency-budget]], [[capacity-estimation]]

---

## 0:13 - 0:21 - Phase 3: High-Level Design

Interviewer: Draw the system. I want to see the offline and the online halves.

Candidate: Two halves. Offline builds, online serves. The online half is aggressively simple because it has to be.

```text
=========================== OFFLINE / BUILD PIPELINE ===========================

  +----------------+   +----------------+   +------------------+
  | Client query   |-->| Query log      |-->| Stream: raw      |
  | events         |   | (Kafka, 3x AZ) |   | queries, 1000/s  |
  +----------------+   +--------+-------+   +--------+---------+
                              |                        |
                              |                        v
                              |              +-------------------+
                              |              | Filter + sanitize |
                              |              | dedupe, blocklist |
                              |              +--------+----------+
                              |                       |
                              |        +--------------+--------------+
                              |        |                             |
                              |        v                             v
                              |  +--------------+           +------------------+
                              |  | Batch:       |           | Stream:          |
                              |  | hourly/daily |           | rolling windows  |
                              |  | MapReduce /  |           | 5m,1h,24h,7d    |
                              |  | Spark        |           | in-memory state  |
                              |  +------+-------+           +--------+---------+
                              |         |                            |
                              |         +-------------+--------------+
                              |                       |
                              |                       v
                              |            +---------------------+
                              |            | Count store         |
                              |            | count-min sketch /   |
                              |            | exact top-K (heavy   |
                              |            | hitters) per prefix |
                              |            +----------+----------+
                              |                       |
                              |                       v
                              |            +---------------------+
                              |            | Trie builder         |
                              |            | top-N per prefix,    |
                              |            | per locale, per      |
                              |            | vertical            |
                              |            +----------+----------+
                              |                       |
                              |                       v
                              |            +---------------------+
                              |            | Snapshot store       |
                              |            | versioned artifacts |
                              |            | in wide-column store |
                              |            +----------+----------+
                              |                       |
       ========== publish new version + atomically swap ==========
                              |
                              v
  +----------------------+                     +-------------------+
  |  Personalization     |  per-user            |  Per-user recent  |
  |  batch job           |-------------------->|  query history    |
  |  last 90 days        |  top-50 prefixes     |  (in user store)  |
  +----------------------+                     +-------------------+

============================= ONLINE / SERVING ================================

   keystroke
       |
       v
  [ Client: debounce 150ms, prefix-length gate, cache LRU 50 ]
       |
       |  GET /v1/suggest?q=iph&limit=10&locale=en-US&vertical=products
       v
  +------------------+     anycast / geo
  |  Edge POP        |-----------------------------------------------+
  |  + local cache   |                                               |
  +--------+---------+                                               |
           | miss                                                      |
           v                                                           |
  +------------------+   HIT   +--------------------------------+      |
  |  Serving tier    |<---------|  L1: in-process trie shard  |      |
  |  (stateless,     |          |  compact double-array,      |------+
  |   per region)    |          |  top-N children per node    |
  +--------+---------+          +--------------------------------+
           | MISS
           v
  +------------------+          +--------------------------------+
  |  Aggregator      |--------->|  L2: memcached / Redis      |
  |  merges tiers,   |          |  top-K per prefix, hot      |
  |  blends personal |          |  prefixes, ~1 TB cluster    |
  +--------+---------+          +--------------------------------+
           | MISS (cold prefix / tail vocabulary)
           v
  +------------------+          +--------------------------------+
  |  Tail lookup     |--------->|  L3: wide-column store      |
  |  prefix scan,    |          |  1e9 queries, sharded by    |
  |  top-K, bounded  |          |  query_hash, 150 GB on disk |
  +--------+---------+          +--------------------------------+
           |
           v
  +------------------+
  |  Response: merge, dedupe, personal blend, safety re-check, top-10 |
  +--------+---------+
           |
           v   200 OK, ~1 KB, X-Cache: L1|L2|L3
        [ Client dropdown ]
```

Candidate: Let me walk it.

Candidate: Query logs. Every keystroke-submitted suggestion and every executed search is written to Kafka, partitioned so that events for one session or one user land together, which matters because personalization needs to join a user's events. Retention is long, 90 days, because the personalization job is a batch job and batch needs raw history.

Candidate: The filter and sanitize stage is a stream consumer doing dedupe, normalization, PII scrubbing, profanity and blocklist enforcement, and language and vertical tagging. It is a stream stage precisely so that a bad query never reaches the count store at all. This is the layer that satisfies the hard correctness requirement about blocked terms, and putting it upstream of counting rather than at request time means the illegal data does not exist in the serving structures.

Candidate: Two counting paths run in parallel. The batch path is a MapReduce or Spark job over the full log that recomputes exact counts and the heavy hitters per prefix. The stream path maintains rolling window counts in memory, roughly one thousand queries a second, so a trend shows up in a five-minute window rather than tomorrow. The stream path is what gives me the five-minute freshness requirement; the batch path is what gives me exactness and the long time windows. Together they are the answer to "batch versus stream", and the correct architecture is both, with the stream path as a low-latency overlay on the batch baseline.

Candidate: The count store. For the top prefixes I keep exact counts because exact top-K matters for ranking. For the long tail, counting every occurrence of a billion distinct queries is a heavy-hitters problem, and a [[probabilistic-data-structures|Count-Min Sketch]] with a conservative update gives near-exact top-K in a fixed small memory footprint, which is exactly the shape of this problem. That is a genuinely good use of a sketch: you do not need exact counts for a query that fires 40 times a year.

Candidate: The trie builder walks the count store and, for every prefix, materializes the top N completions with their scores. N is a key tuning parameter, something like 50, because the client only ever shows 10 but having 50 lets the personalization blend pick from a wider pool, and it makes the structure tolerant to a later change in the scoring function. Building top-N per prefix is 2e10 prefix-entries in the naive full rebuild, which at a few hundred bytes of processing per entry is a large but very parallel job, and it runs on a schedule.

Candidate: Snapshots are versioned and published to a snapshot store, then the serving tier pulls the new version and atomically swaps. That is [[replayability|replayable]]: because the build is a pure function of a versioned log, any snapshot can be rebuilt, diffed, or rolled back, and the build job itself is idempotent because the output is a versioned artifact rather than an in-place mutation.

Candidate: Serving tiers. L1 is an in-process, memory-resident, compact trie shard right inside the stateless serving process, so the hot path is zero network hops. L2 is a shared memcached or Redis tier for prefixes that did not fit in L1. L3 is the wide-column store for the tail vocabulary and for cold prefixes, and it is a bounded scan, never an unbounded one.

Candidate: Personalization is a per-user top-50 prefix list computed nightly and stored against the user, and at request time the serving process checks the requested prefix against that small list. It is a single-key lookup, and it is merged as a boost into the global candidates rather than replacing them, so a user with thin history still gets good global suggestions.

Study separately: [[autocomplete|Autocomplete]], [[mapreduce-lambda-kappa|MapReduce, Lambda, and Kappa]], [[batch-vs-stream-processing]], [[probabilistic-data-structures]], [[replayability]], [[kafka-architecture|Kafka Architecture]]

Interviewer: API contract. And I want the boundary cases handled explicitly.

Candidate: The endpoint.

```http
GET /v1/suggest?q=iph&limit=10&locale=en-US&vertical=products&user_ctx=ab12cd
Accept: application/json
X-Client-Version: 7.4.1

200 OK
X-Suggest-Tier: L1
X-Suggest-Built-At: 2026-01-09T18:08:00Z
Cache-Control: private, max-age=30

{
  "prefix": "iph",
  "suggestions": [
    { "id": "s_8812", "text": "iphone 17 pro", "type": "product",
      "score": 0.98, "source": "global", "highlight_from": 3 },
    { "id": "s_9041", "text": "iphone 17 pro max", "type": "product",
      "score": 0.91, "source": "global", "highlight_from": 3 },
    { "id": "s_3320", "text": "iphone case", "type": "product",
      "score": 0.77, "source": "personal", "personal_boost": 0.34 },
    { "id": "s_1200", "text": "iphone repair near me", "type": "local",
      "score": 0.61, "source": "global" }
  ],
  "took_ms": 3
}
```

Candidate: `highlight_from` is a deliberate optimization, the client already knows the typed prefix, so I ship the byte offset where the completion begins and the client wraps the typed part in a bold span. That removes the need to ship the full query twice and lets the client highlight in one text operation, which is a real latency win on a low-end phone.

Candidate: The `source` field is a product and debugging tool. It tells me whether a suggestion came from global popularity or from the user's own history, and I can use it in the client to de-emphasize personal suggestions if privacy-sensitive users prefer, or simply to analyze whether personalization is helping.

Candidate: Boundary cases, which is where autocomplete systems actually fail in practice.

```text
  q = ""            -> 200, suggestions: [], reason: "prefix_too_short"
  q = " "           -> 200, suggestions: [], reason: "prefix_too_short"  (after trim)
  q = "i"           -> 200, suggestions: [], client shows trending list instead
  q = "zzzzqqq"     -> 200, suggestions: [], reason: "no_match"           (never 404)
  q = "iphone"      -> 200, exact match present, "exact_match": true, allow keyboard shortcut
  q = "a b"         -> 200, space inside query is a valid continuation, not a reset
  q = "cafe" vs "café" -> normalize both to NFD, strip diacritics for the trie key
  q = "iphnoe"      -> 200, no prefix match -> fall back to spell-corrected prefix "iphone"
  q = 5000 chars    -> 413 Payload Too Large, never truncate silently
  q = "..."         -> 429 with Retry-After if the per-client rate limit trips
```

Candidate: Two of these are worth defending. First, zero and one character never produce suggestions, because a single letter has no intent and returning the globally most frequent one-letter prefix is just a popularity contest that always returns the same thing. Instead the client shows trending searches, which is a different, precomputed, entirely cacheable list. Second, no-match returns 200 with an empty array, never 404, because a 404 on a keystroke is meaningless to the client and it means every non-matching prefix logs an error.

Candidate: Debouncing, which is the client-side half of the design and I would raise it unprompted. The client waits 150 milliseconds of inactivity before firing, and it also enforces a minimum prefix length of 2 or 3. Critically, it cancels in-flight requests. When a new keystroke arrives, the previous request is aborted client-side and, if the server supports it, the request carries a monotonically increasing sequence number so the client can discard any response whose sequence is older than the last one it rendered. Without that, rapid typing produces out-of-order responses and the dropdown flickers backwards to an older prefix, which is the single most visible bug in a naive implementation.

Candidate: The server side complements this with a small LRU in the client keyed by prefix, and by noting that a request for a prefix that was served 10 seconds ago is a request I can serve from an edge cache, though for a logged-in user with personalization that cache must be private, never shared.

Candidate: I would also expose the endpoint with a stricter, cheaper path for logged-out and for CDN-cacheable clients, a global-only variant with no personalization, which turns a private dynamic response into a publicly cacheable one and is a very large share of traffic on most products.

Study separately: [[pagination]], [[caching]], [[api-design-principles|API Design Principles]], [[idempotency]]

---

## 0:21 - 0:33 - Phase 4: Deep Dive

Interviewer: The data structure. Walk me through the trie, why not something else, and what it costs.

Candidate: The core structure is a prefix tree, and the reason is that autocomplete is prefix search, and a prefix tree answers exactly that in time proportional to the prefix length plus K, which is independent of vocabulary size. That is the property I need. A hash map cannot do it at all, because hashing "iph" does not find "iphone" unless you happen to have hashed that exact string. A sorted array with binary search gets you there in O(log N) comparisons over 1e9 strings, which is roughly 30 string comparisons and a lot of cache misses, so it is maybe 10 to 50 milliseconds on a cold cache and it does not give you the subtree for free. A finite-state transducer for weighted autocompletion would be the theoretically optimal answer for quality, but it costs a lot of memory and an offline compile step that is hard to re-run, so I would mention it as the quality upgrade, not the starting point.

```text
  Compact double-array / flat array trie

  node := { first_child_idx: uint32, num_children: uint16, flags: uint8, pad: uint8 }
  child_block: contiguous, sorted by character, binary search within block
  node -> suggestion list: head_idx into a separate suggestion array

  +-- "iph" ------ n6  [s8812 iPhone 17 Pro | s9041 ... | s3320 ...]
  |     +-- "one"  (terminal, "iphone")
  |     +-- "one 17 pro" ...
  +-- "iphc" ----- n7
  +-- "ip" -------
        +-- "hone" ...
        +-- "ad"  ...

  Memory budget (top 100M queries, ~250M nodes):
    node array     : 250M x 12 B   =  3.0 GB
    child blocks   : ~300M x 1 B    =  0.3 GB   (packed, inline small blocks)
    suggestion refs: 250M x 2 B    =  0.5 GB   (uint16 index into top-N array)
    top-N array    : 2e9 entries x 8 B = 16.0 GB  -> prune to top-20 = 6.4 GB
    string table   : 100M x 24 B    =  2.4 GB
    ----------------------------------------------------------------
    hot working set                          ~= 12 GB  (pruned, compacted)
```

Candidate: The memory discipline is the whole trick, and there are three decisions in it. One, a flat array instead of an object graph, because 250 million heap objects with 200-byte headers is 50 gigabytes before you store anything, and an array of 12-byte structs is 3 gigabytes. That is a 15x difference purely from representation. Two, suggestion text is not stored in the trie, only a 2-byte index into a string table, so the trie stays small and the text is shared. Three, N is pruned. The naive build keeps top-50 per prefix and that is 16 gigabytes of suggestions alone; top-20 is 6.4. The client shows 10, so the extra 10 is the personalization pool, which is a cheap insurance policy against a scoring change.

Candidate: Sharding the trie. I shard by the first one or two characters of the prefix, and this is a nice case where a naive hash of the query is wrong, because I need all queries sharing a prefix on the same shard, otherwise a prefix query becomes a scatter-gather across every shard. Hashing the first character gives at most a few hundred shards for ASCII, which is more than I need, so I use a two-level scheme: hash the first character into one of about 64 buckets, and within a bucket keep a sorted array of the full prefixes, so the first character narrows it and a binary search finishes it. Or, with [[consistent-hashing|Consistent Hashing]] on the first two characters, I get a more even distribution when the distribution is skewed toward certain letters, with [[virtual-nodes|virtual nodes]] so I can grow a hot bucket later.

Candidate: The consequence of prefix sharding is that load is not uniform, because "a" is a hundred times hotter than "z". That is fine and I handle it by giving the hot buckets more serving replicas, which is a very cheap form of [[sharding|sharding]] where the shard key is the query prefix itself.

Candidate: Top-K extraction. When I serve a prefix, I walk the node, and I need the top K across that node's terminal suggestions plus all its descendants. I precompute and store the top-N for every node at build time, which is what the 2e9-entry table is. So at request time it is an array read and a partial sort of 20 items, which is sub-microsecond. The alternative, walking the subtree at request time, is out of the question, because a popular prefix like "a" has millions of descendants.

Candidate: Ranking, and this is where product and engineering meet. The score for a completion given a prefix is a blend.

```text
  score(candidate, prefix, user, now) =
      w_p * log1p( count_5m(prefix + candidate) )      # recency-weighted volume
    + w_h * log1p( count_24h(prefix + candidate) )
    + w_t * log1p( count_7d(prefix + candidate) )
    + w_e * log1p( count_ever(prefix + candidate) )   # evergreen, low weight
    + w_u * affinity(user, candidate)                  # personal history
    + w_c * completion_quality(candidate, prefix)      # length, dict status, vertical match
    - w_d * diversity_penalty(already_shown_candidates)

  personal blend:  if candidate in user's top-50 prefixes, add personal_boost
                   else score unchanged.  NEVER replace the global list.
```

Candidate: Four properties I want to defend. First, all counts are log-compressed, because raw counts make a single dominant query win every slot, and a dropdown where all ten rows are the same brand is a bad dropdown. Second, time windows are blended rather than switched, so a spike rides the 5-minute weight while an evergreen query rides the 7-day and ever weights, which is exactly the right behavior. Third, popularity is decayed, not deleted, so a query that dies keeps a small residue and can resurface. Fourth, personalization is a boost on top of the global list, never a replacement, because a user with 12 searches of history must not get a 12-item suggestions list, and because replacement leaks the private signal into a globally-shaped response.

Candidate: Personalization in detail. The signal is the user's own recent query history, so the pipeline is: a nightly batch job reads the last 90 days of that user's query events, counts queries per prefix bucket, keeps the top 50 prefixes and the top 3 completions under each, and writes one small record. Size: 50 prefixes times 3 completions times roughly 40 bytes is about 6 kilobytes per user. At 500 million users that is 3 terabytes, which is a real store, so I shard it by user and it is a single-key lookup at request time, usually an L2 or Redis hit. I could also do a much cheaper version, just the last 20 raw queries, which is under a kilobyte and captures most of the signal. I would ship the cheap version first and measure.

Candidate: The privacy boundary. The per-user record is fetched by user id with a strict authorization check, it is never joined with another user's, it has a retention window, and there is a user-facing delete-my-search-history that invalidates the record. And the response tells the client which rows were personalized via the source field, so a privacy-conscious user can see exactly what personalization is showing them.

Interviewer: Precomputation. Go deeper on the pipeline and on how you rebuild without downtime.

Candidate: The build is a pure function of a versioned log slice, which is the property that makes everything else safe. Output is a new immutable snapshot identified by build_id. Nothing is ever mutated in place.

Candidate: Three cadences, each producing a different thing. The 5-minute stream job updates the hot window counts and a small hot-prefix trie delta, this is the freshness overlay. The hourly job recomputes the full trie with the freshest windows for the top prefixes. The daily job does the authoritative full rebuild over the whole 1e9 vocabulary, produces the long-window counts, and emits the new snapshot version. The daily job is the one that takes hours of cluster time; the hourly one is incremental and only touches prefixes whose counts changed, which is a small fraction.

Candidate: Hot swap without downtime. The serving tier keeps the current snapshot_id in memory and a background refresher polls for newer versions. It downloads the new snapshot to a local memory buffer, builds the trie off the serving path or in a forked child process, validates it, and then flips a single atomic pointer. In-flight requests keep using the old snapshot because they captured the pointer. So the swap is a pointer assignment, and there is no window where a request sees a half-built trie. Rollback is the same mechanism pointing at the previous snapshot_id, which takes seconds and is the reason I insist on versioned artifacts.

Candidate: Validation before swap is not optional. I run automated checks: prefix-continuation correctness, that every returned suggestion actually starts with the requested prefix, that no blocklisted term is present, count monotonicity against the source of truth on a sample, and a latency microbenchmark. A snapshot that fails validation is never published, and the build job alerts rather than silently serving yesterday's data.

Candidate: Replayability. Because the input is an immutable log, I can rebuild any historical snapshot for debugging, which is huge when someone reports "the suggestions were wrong at 3pm". I can reproduce the exact input slice and re-run the build deterministically if I fix the random seed in the sketch and the tie-breaking. I also replay the stream when a bug lands in the sanitizer, by rewinding the topic to the affected offset and re-running, which is only possible because the stream stage is a pure transformation with external state that I can snapshot.

Candidate: Consistency between the windows. There is a real subtlety: the stream overlay and the batch baseline are two different computations and they can disagree. I do not try to merge them into one number, I keep them as separate features in the ranking function with separate weights. That is the cleaner design, because merging two count sources into one is where reconciliation bugs live.

Study separately: [[search-ranking]], [[data-warehouse-lake]], [[schema-migration|Schema Migration]], [[kafka-retention|Kafka Retention]], [[observability]]

Interviewer: Follow-up one. Why not just use Elasticsearch for this? It already does prefix queries, it already has analyzers, why build a custom trie?

Candidate: The honest answer is that Elasticsearch is a strong choice and many production systems do exactly this with it. My reasons to go custom are specific, not ideological.

Candidate: One, latency and shape of the work. An ES prefix query is either a `prefix` query over an analyzed field or a `fuzzy` or `edge_ngram` trick. The `edge_ngram` approach, indexing "iph" to "iphone" as a token, is the classic way and it works, but it explodes the index size, roughly by the average token length, so 20 characters a token means up to 20x the term count, and I am already at 1e9 terms. On a 20-node cluster, a prefix query that touches a hot shard has a p99 in the tens of milliseconds, and under a cache miss storm during a trend it degrades. My trie is a single array read with a predictable cost.

Candidate: Two, memory economics. A custom compact trie is 12 gigabytes for the hot 100 million queries. The equivalent ngram index in a search engine is 40 to 100 gigabytes of heap and it competes with the actual search results index for the same nodes. For an endpoint that is 99.9 percent reads of a tiny derived result set, spending the primary search cluster's memory on it is the wrong allocation.

Candidate: Three, I do not want the suggestion traffic to be able to take down search. Same failure domain otherwise. An autocomplete outage triggered by a trend taking down the results page is a much worse incident.

Candidate: Where I would use Elasticsearch is the tail, the L3 tier, where I want fuzzy matching and edit distance for the typo case, and offline for the build, where an analyzer gives me normalization and tokenization for free. So my real answer is: search engine for the build and the long tail, custom compact trie for the hot path.

Interviewer: Follow-up two. Why not compute suggestions on the fly with a `LIKE 'iph%'` query? Why precompute at all?

Candidate: Because of the shape of the read. A `LIKE 'iph%'` with an index is either a range scan that is fine on a small table or a full scan, and on a 1e9-row table under 1.2 million requests per second it collapses immediately. Even with a perfectly good index it is a disk or page-cache round trip and a scan of the matching rows, and the number of matching rows for a popular two-character prefix is in the millions. So on-the-fly prefix search is O(matching rows), which is unbounded, whereas a precomputed top-N is O(prefix length + K), which is constant. That asymmetry is the entire argument for precomputation.

Candidate: The other cost is compute. Deriving top-N for a prefix requires aggregating counts for all completions, which is a group-by over the whole subtree. Doing that per request, at 1.2 million requests a second, is a different order of magnitude of CPU than reading a precomputed array. Precompute trades memory, which is cheap and bounded, for CPU, which is the scarce resource at peak.

Interviewer: Follow-up three. A worldwide event. Twenty million people search "result" within an hour and a brand-new query goes from zero to a top-ten trend. What happens in your system?

Candidate: Let me split it into what works and what breaks.

Candidate: What works: the stream path sees the spike within a five-minute window and the 5-minute window count for the relevant prefixes climbs. Ranking is a blend, so the new query gets a high 5-minute weight and, with log compression, a high but not absolute score. So a genuinely trending query does surface within about five minutes, which is the requirement. And because the suggestion list is top-10 with a diversity penalty, one exploding query does not take all ten slots.

Candidate: What breaks, and there are three real problems. One, the top-N tables for hot prefixes were built assuming a stable distribution. A prefix whose subtree explodes now has a top-20 that is stale relative to a five-minute-old count store. The hourly rebuild fixes it, but I want better than hourly for exactly this case, so I add a targeted fast path: the stream job detects that a prefix's 5-minute count has crossed a multiple of its daily count, meaning something is happening that the baseline has never seen, and it emits a hot-prefix rebuild request for that prefix subtree only. Rebuilding one subtree is cheap and bounded.

Candidate: Two, the cache stampede. If twenty million people all type the same letters at the same time, and the serving tier is missing for those prefixes, that is a 1.2 million per second miss storm against L3. Defenses: a negative-cache entry for known-cold prefixes so L3 is hit at most once every few seconds, request coalescing so that concurrent identical prefix lookups collapse to one, and a circuit breaker on L3 that serves a degraded, smaller, or trending-only result rather than queueing. And critically, a pre-warmed L1 for the top few hundred thousand prefixes, because the distribution is so concentrated that the top hundred thousand prefixes might be 80 percent of all traffic, which means most requests are never a miss.

Candidate: Three, and this is the one that would actually page me: the snapshot build cost. A daily full rebuild over a vocabulary that just grew by ten million trending queries takes longer than its window, and I get a build backlog. The defenses are an incremental build for changed prefixes only, a size guard that fails the build rather than pushing an oversized artifact, and a rollback to the previous snapshot. The important design property is that the build failing does not affect serving at all, because the serving tier keeps serving the last good snapshot. That separation is what keeps a pipeline problem from becoming an outage.

Candidate: A fourth, more subtle one: blocklist propagation. If a trending query turns out to be unsafe and I add it to the blocklist, it is already in snapshots that are being served, and in client-side caches. So the blocklist check must exist on the serving path as a fast in-memory set lookup in addition to being applied in the build, otherwise a safety violation persists for up to a snapshot cycle. I would rather pay one hash lookup per returned row than have a compliance gap.

Interviewer: Follow-up four. Edge and regional placement. Where does this actually run?

Candidate: Three tiers of placement. First, the edge. The global-only, logged-out, personalization-free variant is a public response and it goes behind the [[cdn|CDN]] with a short TTL, and it is by far the largest share of traffic, so it is the cheapest thing to serve and it absorbs casual and bot traffic. The private, personalized variant cannot be shared-cacheable, so it does not go to a shared CDN, though it can go to a regional POP with a private cache.

Candidate: Second, the regional serving tier. Multiple independent deployments, one per region, each with its own L1 trie shards, each with its own L2, each able to take 100 percent of its region's traffic. Users are routed by [[geo-dns-anycast|Geo DNS and Anycast]] or locality-based routing to their nearest region. Because suggestions are derived data with a five-minute staleness budget, each region builds and serves its own snapshot from its own log slice, or serves a replicated snapshot. Regional independence is free here in a way it would not be for a transactional system, and that is a real benefit of the read-only, rebuildable design.

Candidate: Third, and this is a nice trick, the client. The client keeps an LRU of the last several hundred prefix responses, and for the overwhelmingly common case of a user typing a query they have typed before, or a continuation of one, the answer is already on the device. So a meaningful fraction of keystrokes never hit the network at all, and the server is only consulted for prefixes the client has not seen. On a bad network this is the difference between a working dropdown and an empty one.

Candidate: What I would not do is put a personalizable trie in a globally shared cache, because that is a data leak. Personalization is per user, and per-user data in a shared cache is one hash collision away from being served to the wrong person.

Interviewer: Follow-up five. Storage. Justify keeping this in memory at all. And what about non-Latin scripts?

Candidate: Storage justification. I keep in memory only the top 100 million queries and only their prefixes, which I sized at about 12 gigabytes compacted, and that is affordable at roughly 2,000 to 4,000 dollars a month of RAM. The full 1e9 vocabulary at 150 gigabytes lives on disk in a wide-column store and is the fallback, because a query in the tail 900 million is rare enough that a 5 to 20 millisecond disk lookup is acceptable for it, and the top-K for those is small anyway. The economic argument: memory for the top 1 percent because it serves the top 99 percent of traffic, disk for the rest. That is just [[storage-tiering]] applied to a derived index.

Candidate: Scripts and locales. Three problems and three answers. One, size. A character in CJK is roughly three bytes where an ASCII character is one, and Chinese queries are short, four to six characters, so per-query memory is comparable but prefix fan-out is different because there are fewer distinct characters. Two, normalization. "café" typed as e plus a combining accent versus as é are different byte strings, so I normalize to a canonical form, NFD or NFC consistently, at the sanitizer stage, and I store both a normalized key and a display string so the client shows the accented form while the trie matches the normalized one. Three, case and script folding, Turkish dotless i is the classic trap, so I do locale-aware folding at build time using the query's own locale, not the server default. And each locale has its own trie shard, because merging them wastes memory on a tree that no locale will traverse.

Interviewer: Follow-up six. How do you know it is working? What is your quality and reliability bar?

Candidate: Quality, offline first. I take a sample of prefixes, generate candidates from each tier, and compute the right metric for this problem, which is not NDCG exactly but precision or recall at K against the query that users actually went on to execute. Concretely: for a sample of real keystroke sessions, for the prefix the user typed at position n, what fraction of my top 10 contains the query they actually executed? That is the metric that maps to the product outcome, and it is very sensitive to the debounce and prefix-length choices, which is a pleasant property because those are the knobs I most want to tune.

Candidate: I also track suggestion acceptance rate, which is the click-through on a suggestion divided by suggestion impressions, broken out by whether the suggestion was global or personal, which is how I prove personalization is earning its 3 terabytes. And a coverage metric, the fraction of executed queries that appear as a suggestion anywhere, which is the number that tells me whether the tail vocabulary is being served well.

Candidate: The negative metrics matter as much. Zero-result rate, which catches a build pipeline that stopped ingesting, and a coverage-drop alert. And a "same ten rows" detector, which catches a bug where ranking collapsed and every prefix returns the same popular set, which is a silent failure that no latency or error metric would ever show you.

Candidate: Reliability. This is a read-only serving system, so the SLOs are availability and latency, and the error budget should be spent on p99 not p50. I track prefix-query p99 per tier, L1 hit ratio per shard, L2 hit ratio, L3 query rate, snapshot age, which is the freshness SLO, and build freshness, which is the offline SLO. Snapshot age is the metric I would put on the executive dashboard, because a stale suggestion set is functionally an outage that returns 200s and looks healthy.

Study separately: [[tail-latency|Tail Latency]], [[sli-slo-sla|SLIs, SLOs, and SLAs]], [[edge-computing]], [[load-shedding]], [[bulkhead|Bulkhead]]

---

## 0:33 - 0:43 - Phase 5: Trade-offs and Follow-ups

Interviewer: Trade-offs. I want them stated, not implied.

Candidate: Eight, in order of how much they cost me.

Candidate: One, I accept approximate counts in the tail. A [[probabilistic-data-structures|Count-Min Sketch]] over a billion queries is not exact, so a tail query's count can be over-estimated. What I buy is a fixed, small memory footprint and O(1) updates, which is the only reason I can count a billion distinct queries at a thousand updates per second. Mitigation: I use a conservative update and a wide sketch, which makes over-estimation small, and I use exact counts for everything in the hot set where exactness actually affects ranking.

Candidate: Two, I accept a fixed top-N per prefix. A query that is the 21st most popular under a prefix can never be suggested for that prefix, even if the user has searched for it. What I buy is constant-time top-K and a bounded memory table. Mitigation: the N is a build parameter, so raising it is a config change and a rebuild, and the rebuild is cheap. The right way to detect the cost of this choice is to measure, for executed queries that did not appear in suggestions, their rank under the prefix. If many are rank 15 to 25, raise N.

Candidate: Three, I accept a precomputation lag of up to five minutes for new trends. I chose that over a real-time path because a real-time suggestion path means a group-by on the read path, which is the thing I proved I cannot afford at 1.2 million requests per second. Mitigation: the targeted hot-prefix rebuild makes the common viral case much faster than five minutes in practice.

Candidate: Four, I accept memory over cost, twelve gigabytes plus three terabytes of personalization, and I could cut the personalization tier to twenty recent queries per user for a kilobyte each if the budget tightened.

Candidate: Five, I accept that personalization is a coarse signal. Top prefixes per user, not a learned per-query model. A learned ranking over per-user features is the quality upgrade and I would do it, but it moves personalization from a single key lookup to a feature-join, and I would only pay that after measuring that the coarse version is the bottleneck on acceptance rate.

Candidate: Six, I accept regional independence over global consistency. Two users on two continents can see different suggestions for the same prefix for up to a few minutes. For this product that is right, because a consistent-but-500-millisecond dropdown is a worse product than a consistent-with-5-minutes-and-8-millisecond dropdown.

Candidate: Seven, I accept a hand-tuned ranking function as the starting point over a learned ranker, for the reason that explainability and attribution matter enormously when suggestions are visibly wrong to a user, and because a learned model over these features needs an offline training loop I would rather not build before the basics are right.

Candidate: Eight, I accept that the blocklist is enforced in two places, the build and the serving path, which is duplicated logic and therefore a drift risk. I buy the guarantee that a newly blocked term cannot be served from an already-published snapshot. Mitigation: a single shared library and a canary check that both paths agree on a test corpus.

Interviewer: Rapid-fire failure scenarios.

Interviewer: The stream job dies and nobody notices for two hours.

Candidate: Serving is unaffected, because the daily snapshot is still being served, so this is a freshness degradation and not an outage. But suggestions get visibly stale exactly when they matter, during a trend. This is caught by the snapshot-age and window-age alert, and by the zero-result-rate and coverage metrics moving the wrong way. The runbook is restart the consumer from its last committed offset, which works because the log is retained for 90 days. The structural improvement I would want is a consumer-lag alert with a threshold in the low minutes, not an error-rate alert, since there is no error.

Interviewer: The daily build produces a snapshot that is 40 percent smaller than yesterday, and nobody knows why.

Candidate: That is the most dangerous class of build bug, because the artifact is valid and the service stays up while the product quietly degrades. Defenses, in order: a hard size-change guard that fails the build at more than a 20 percent delta in either direction; a coverage metric comparison, suggestion coverage must not drop more than a set percentage; a sampling validation where a sample of prefixes' top-10 is diffed against the previous snapshot and the diff is reported to the build log, so a mass disappearance is visible even when the artifact is technically valid; and never publish a snapshot that fails validation, keep serving the last good one. The principle is that the read path must be able to survive a bad artifact indefinitely, which is what makes the pointer swap safe.

Interviewer: A single trie shard is slow. The "a" bucket is 3x the latency of the others.

Candidate: That is a hot shard, and it is expected because prefix distribution is Zipfian. I handle it with more replicas for the hot bucket, which is cheap, and I can split a bucket further by hashing the second and third characters within it, so "a" splits into "aa" to "am", "an" to "az" and so on, at the cost of a slightly wider fan-in. I also keep the small optimization that the first character alone usually narrows it enough, and I use a hint in the client cache. If a single prefix, not bucket, is pathological, the per-process LRU absorbs it, and a per-shard circuit breaker stops it from spreading.

Interviewer: A crawler or bot sends every possible two-letter prefix at maximum rate to map your vocabulary.

Candidate: That is prefix enumeration and it is a real attack. Defenses: the minimum prefix length of 2 or 3 does not help because two letters is exactly what they are enumerating, so I rate limit per client and per IP with a token bucket at the edge, with a much tighter bucket on the L3 path than on L1, since a legitimate high-QPS client almost always hits L1 or the CDN. I also put a hard cap on the number of distinct prefixes a single client may request per minute, which specifically targets enumeration rather than volume. And I keep the negative cache, so the enumeration costs me one L3 lookup per prefix per cooldown period rather than one per request. A bot that is enumerating gets served the tail, which is low-value to it, and eventually gets shaped.

Interviewer: Personalization store is down. What do users see?

Candidate: Global suggestions, and nothing breaks. The serving process treats a personalization lookup failure as a miss, not an error, so the response is a normal 200 with the global list. This is deliberate, because personalization is an enhancement, and coupling the availability of the primary function to an optional enhancement is a design error. I do get a metric, personalization-lookup-error-rate, and a drop in acceptance rate that I would otherwise not see. The only thing I lose is the ability to ship a feature on top of the personal store, so I'd want a graceful degradation mode there rather than a hard dependency.

Interviewer: Region goes down entirely.

Candidate: Geo DNS fails over to another region. That region has its own complete serving stack, its own L1, L2, and its own snapshot, so it serves its regional data at full quality with no cold start. If the region was a follower of a global snapshot, it promotes and serves a slightly older snapshot, which is a five-minute cost, not an outage. The one thing that does not fail over is the build, if the build ran only in the dead region, which is why I run the build redundantly in two regions and the snapshot store is cross-region replicated. Note the asymmetry: serving is trivially available because it is read-only and derived, and the build is the fragile part, which is the opposite of the usual intuition and worth stating explicitly.

Interviewer: You said stale-while-revalidate. Where does that apply here, since you precompute?

Candidate: In exactly one place, the L2 and L3 tiers and the L1 refresh. A stale top-N list is a perfectly good suggestion list, since the client's own next keystroke changes the prefix anyway, and users do not notice a top-N list that is 60 seconds old. So I serve stale immediately and refresh in the background, which removes the stampede problem almost entirely. The one place I would not do it is the blocklist, because there staleness is a compliance issue, and the blocklist lookup is an in-memory set on the serving path anyway so it is never stale.

Interviewer: If I gave you a week, what is the highest-leverage change?

Candidate: Event-driven per-keystroke feedback, closed loop. Right now popularity is a pure function of executed queries, which lags the user by a full session. If I log suggestion impressions and which suggestion was accepted per keystroke position, I can feed that back into the ranking as a much denser signal, and more importantly I can answer the question of whether a suggestion helped, not just whether a query was popular. That also unlocks proper offline evaluation with a real counterfactual, because I can hold out suggestions and measure acceptance. Second highest would be the weighted finite-state transducer for the quality upgrade, and third would be a learned reranker over the per-user features. But the FST and the model are quality upgrades on top of a pipeline; the feedback loop is what makes every future improvement measurable, and right now I cannot measure whether personalization helps except through a blunt acceptance-rate comparison.

Study separately: [[load-shedding]], [[circuit-breaker|Circuit Breaker]], [[multi-region-models]], [[cross-region-replication]], [[bft|Consensus]]

---

## 0:43 - 0:45 - Final Architecture Summary

Interviewer: Two minutes. Summarize.

Candidate: I built a two-half system. The offline half is a pipeline: query logs go to Kafka, a stream stage sanitizes, normalizes, blocklists and tags them so illegal queries never enter the counting layer, then a stream path maintains 5-minute, 1-hour and 24-hour rolling counts while a MapReduce job does the authoritative daily computation over the full billion-query vocabulary using exact counts for the hot set and a Count-Min Sketch for the tail. A trie builder walks the counts and materializes the top 20 completions and their scores for every prefix, per locale and per vertical, and publishes that as an immutable, versioned snapshot. Because the build is a pure function of a versioned log, any snapshot is replayable, diffable, and rollback-able.

Candidate: The online half is a read-only, latency-optimized serving path. The client debounces at 150 milliseconds, gates on prefix length, aborts stale in-flight requests with a sequence number, and keeps a small on-device LRU so repeated typing never hits the network. The request goes to a regional serving tier over anycast, where the hot corpus lives as a compact double-array trie in process memory, roughly 12 gigabytes for the top 100 million queries, sharded by query prefix so a lookup is one array read and a partial sort of 20 items. A miss goes to a memcached or Redis tier, then to a sharded wide-column store for the tail, with coalescing, negative caching, and a circuit breaker. Personalization is a 6-kilobyte per-user top-prefix record that acts as a score boost on top of the global candidates, never a replacement, with a blocklist re-check on the serving path as a hard correctness guarantee.

Candidate: The three decisions that carry it: suggestions are precomputed top-N per prefix because prefix search is O(prefix plus K) when precomputed and O(matching rows) when computed live, and only the first survives 1.2 million requests per second. Serving is a read-only derived artifact, which is why regional failover is free, why a bad build cannot take down serving, and why the whole hot tier is a 12-gigabyte in-memory structure. And the pipeline is a pure function of an immutable log, which is what makes the five-minute freshness goal, the atomic snapshot swap, and replay-based debugging all possible at the same time.

Candidate: What I accept in exchange: approximate tail counts, a fixed top-N cutoff, up to five minutes of trend lag, a memory bill, per-user personalization that is coarse and 3 terabytes wide, and regional rather than global consistency. Each of those is freshness or exactness traded for latency and cost, which is the correct currency for a keystroke-latency surface. If the requirement changed to exact ordering, sub-second trend reflection, or globally consistent suggestions, I would re-open the design, and I would move to real-time per-request aggregation with a much larger compute bill and a latency profile that no longer fits 100 milliseconds.

Interviewer: Good. That's the time.

---

## Coverage Checklist

- Requirements clarification: prefix-length gate, debounce, latency budget, top-K contract, ranking blend, staleness tolerance, privacy
- Scale, traffic, and storage estimation: keystroke multiplier, QPS, egress bandwidth, vocabulary, node count, memory budget, precomputation volume
- API design: GET /v1/suggest contract, highlight offsets, source attribution, ten boundary cases
- High-level architecture: ASCII diagram of offline build and online serving
- Request and data flow: keystroke to debounce to edge to tier to response
- Data structures: compact double-array trie, node layout, memory budget table, sharding by prefix
- Top-K extraction: precomputed per-node top-N, sub-microsecond read
- Ranking: blended log-compressed windows, diversity, personal boost
- Personalization vs global: boost not replace, per-user record, privacy boundary
- Caching: L1 in-process, L2 shared, L3 wide-column, client LRU, CDN for global variant
- Precomputation: MapReduce batch, stream windows, build cadence, atomic swap, validation
- Replayability: pure function of log, deterministic rebuild, rewind and replay
- Debouncing: client-side, min prefix length, abort, sequence numbers
- Boundary cases: empty prefix, single char, no match, exact match, spaces, diacritics, typos, length limit, rate limit
- Edge and regional placement: CDN for public, regional tiers, anycast, client-side, why not shared-cacheable personal data
- Trade-offs: eight explicitly accepted
- Failure scenarios: stream job death, undersized snapshot, hot trie shard, prefix enumeration, personalization outage, region loss, stale-while-revalidate scope
- Follow-ups: why not Elasticsearch, why not on-the-fly LIKE, viral trend, edge placement, storage justification, non-Latin scripts, metrics, one-week priorities
- Final summary: three-part close
