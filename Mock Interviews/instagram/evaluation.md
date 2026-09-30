---
title: Instagram - Interview Evaluation
status: active
date: 2026-09-29
tags: [hld, mock, instagram, evaluation]
---

# Instagram - Interview Evaluation

## Scoring

Scored out of 5 per phase. 4 or above on every phase is a pass; a 5 on Phase 2
or 4 with a design that still hangs together is what a senior candidate looks
like.

| Phase | Score | Why |
| --- | --- | --- |
| Requirements and Constraints | 4 | Split the like question into "is the count exact" versus "is my like state exact", which is the exact distinction the design turns on, and explicitly rejected algorithmic discovery when scoped out. Lost a point for not asking about private accounts or a web client before designing. |
| Scale Estimation | 5 | Wrote out the arithmetic, and critically pushed back on a stated assumption of 150M videos per day as implausible before designing around it. Landed the insight that read-to-write ratio is roughly 1:1, not read-heavy, because likes are writes. |
| API Design and Data | 4 | Presigned direct upload with idempotency token, opaque cursor instead of offset, and the "PUT because like is idempotent" reasoning are all strong. Dedup defence given as three independent layers rather than one mechanism. |
| Architecture and Data Flow | 5 | Diagram is legible, every arrow is justified, upload path provably bypasses application servers, and the outbox pattern was used specifically to fix the dual-write failure rather than mentioned decoratively. |
| Deep Dive and Failure | 5 | Hybrid fanout with an explicit threshold, sub-sharded counters, fencing called out by name during failover, and a real answer to data erasure that included the anonymous-tombstone product decision. |

Total: 23 / 25

## What Made This a Strong Answer

- **Pushed back on the numbers before using them.** Treating a stated
  assumption as a hypothesis to validate rather than a fact to design around is
  the single highest-leverage habit in the whole interview, and it happened in
  minute three.
- **Found the real asymmetry.** The read-to-write ratio being about 1:1
  contradicts the standard mental model of a social app, and reframing the
  problem as "reads are cheap, writes have fanout cost" is what made the rest of
  the architecture follow logically instead of being assembled from patterns.
- **Named the exact mechanism for every failure.** Not "we would use
  distributed locks" but "a lease with an epoch number, checked on every write,
  to prevent split-brain on failover". Precision is what separates a senior
  answer from a plausible one.
- **Every consistency claim was per-relation, not per-service.** Saying "the
  like service is eventual" says nothing; saying the like service holds three
  relations with three different guarantees is a design.
- **Gave the hot creator a threshold and a cost model**, then said the threshold
  would have to be re-tuned and explained what the new tuning criterion would be.
  Volunteering that your own magic number is temporary is a strong signal.
- **Answered the deletion question as a product and legal decision, not just a
  purge job**, and chose the reversible option (anonymized tombstone) over the
  irreversible one for other users' content.

## Memory Hooks

- Media never touches your servers. Presigned multipart, bytes go straight to
  the object store, full stop.
- 3.5 TB a day ingested, 64 GB a second of egress, 90 percent of it absorbed by
  the CDN. The CDN is not an optimization here, it is the difference between
  5 Gbps and 500 Gbps.
- One point five kilobytes of metadata per post against three megabytes of
  photo. That three-orders-of-magnitude gap is why media and metadata live in
  completely different systems.
- Read-to-write is 1:1 because likes are writes. "Social apps are read-heavy"
  is wrong and it will mislead your whole storage design.
- Count is eventual, your own like is strong. Split the requirement before you
  split the schema.
- Fanout on write under 10K followers, pull on read over. The hot creator is one
  extra cached sorted-set read, not 500 million writes.
- Sub-shard the hot key 32 ways. Writes get better, reads get thirty-two-way
  fan-in, and for likes that is the right direction to trade.
- Fencing token on failover, or you have two writers and no error message.

## Weak-Spot Pointers

- **The threshold number itself was asserted, not derived.** 10,000 followers was
  presented as a constant when it is a function of fanout write capacity, post
  rate, and read cost. Drill [[shard-rebalancing|Shard Rebalancing]] and
  [[horizontal-vs-vertical-scaling|Horizontal versus Vertical Scaling]] so you
  can defend the number with arithmetic instead of intuition.
- **Read-your-writes for follows was identified as a bug and then patched with
  hand-waving.** "Immediate synchronous on-read pull" and "hot followee
  priority" were both asserted without a mechanism. Drill
  [[strong-vs-eventual-consistency|Strong versus Eventual Consistency]] and
  [[replication-lag|Replication Lag]] and be able to name the token or the
  routing rule that closes the window.
- **Sharding strategy was stated per table but the resharding mechanics were
  compressed into one sentence.** Dual-write, backfill, flip is the right
  skeleton, but the risky step is the backfill plus delta catch-up. Drill
  [[sharding|Sharding]], [[shard-routing|Shard Routing]] and
  [[data-migration|Data Migration]].
- **Cache stampede got five mitigations and no cost model.** You listed jitter,
  stale-while-revalidate, single-flight, logical expiry and warming without ever
  saying which one you would ship first or what each costs in memory or in
  staleness. Drill [[cache-warming|Cache Warming]] and
  [[hotspot-handling|Hot Key Handling]] and quantify the trade.
- **Video playback specifics were hand-waved as "DASH or HLS".** If an
  interviewer goes one level deeper on the video pipeline, adaptive bitrate
  manifests, segment sizing and the difference between VOD streaming and live
  are all live topics. Drill [[media-processing|Media Processing]] and
  [[blob-storage|Blob/Object Storage]] for the rendition and lifecycle half.
- **No cost estimate was given.** You had every input needed for a
  cost-estimation [[capacity-estimation|Capacity Estimation]] and skipped it.
  Storage tiers and egress are the dominant lines in this business and saying
  "egress dominates cost" without a number is a weaker answer than you could
  have given.

## Read Next

- [[06-hld-interview-checklist|HLD Interview Checklist]] - run the five-phase
  gate against this transcript and confirm nothing was skipped
- [[01-rapid-revision|Rapid Revision]] - the consistency table and the feed
  model comparison are the two things worth having in cold memory
